# Ejercicios de minimal APIs: Typed Results y endpoint filters

Cuaderno de práctica basado en los ejercicios de la Sesión 10, **revisados y corregidos**. Mantienen los nombres en inglés de la clase (`Product`, `Order`, `FakeDb`). Cada solución es el `Program.cs` completo de un proyecto `dotnet new web`.

Requisitos previos: las tres lecciones del [módulo](README.md).

Todas las soluciones usan esta base de datos falsa; pégala al final de cada `Program.cs`:

```csharp
public record Product(int Id, string Name, decimal Price);
public record Order(int Id, int CustomerId, decimal Total);

public static class FakeDb
{
    public static readonly List<Product> Products = [new(1, "Keyboard", 150m), new(2, "Mouse", 60m)];
    public static readonly List<Order> Orders = [new(1, 10, 210m), new(2, 20, 80m)];
}
```

(Las listas estáticas no son seguras con peticiones concurrentes; sirven para practicar.)

-----

## Ejercicios guiados

### Ejercicio 1: Typed Results

**Objetivo:** un endpoint con varias respuestas declaradas en la firma.

**Contexto:** el producto puede existir o no.

**Instrucciones:**

1. `GET /products/{id:int}` con `Results<Ok<Product>, NotFound>`.
2. Escribe el handler como método con nombre, para poder probarlo.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
var app = builder.Build();
app.UseStatusCodePages();

app.MapGet("/products/{id:int}", ProductHandlers.GetById);

app.Run();

public static class ProductHandlers
{
    public static Results<Ok<Product>, NotFound> GetById(int id)
    {
        var product = FakeDb.Products.FirstOrDefault(p => p.Id == id);
        return product is not null ? TypedResults.Ok(product) : TypedResults.NotFound();
    }
}

// + Product y FakeDb
```

</details>

**Qué observar:**

* La guía decía que el compilador "optimiza la serialización" y que se "evita boxing". No es así: `Results.Ok` y `TypedResults.Ok` crean el mismo objeto (una clase, sin boxing) y se serializa igual. Lo que ganas es que **devolver algo no declarado no compila**, que **OpenAPI documenta 200 y 404 solo** y que el handler se prueba directamente:

  ```csharp
  Assert.IsType<NotFound>(ProductHandlers.GetById(999).Result);
  ```

* `{id:int}`: `/products/abc` da 404 en lugar de 400.

-----

### Ejercicio 2: validar una API key con un filtro

**Objetivo:** cortar la petición antes del handler si la API key no es válida.

**Contexto:** la guía leía el header en `key` pero **nunca lo comparaba**: cualquier valor pasaba. Y la clave correcta no estaba en ningún lado.

**Instrucciones:**

1. Guarda la clave en la configuración (`appsettings.Development.json`: `"ApiKey": "dev-key-123"`).
2. El filtro compara el header `X-Api-Key` con esa clave, en tiempo constante.
3. Si falla, 401 con ProblemDetails.

<details>
<summary>Solución</summary>

```csharp
using System.Security.Cryptography;
using System.Text;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
var app = builder.Build();

app.MapGet("/data", () => "Información sensible")
   .AddEndpointFilter(async (context, next) =>
   {
       var config = context.HttpContext.RequestServices.GetRequiredService<IConfiguration>();
       var expected = config["ApiKey"] ?? throw new InvalidOperationException("Falta 'ApiKey'.");

       if (!context.HttpContext.Request.Headers.TryGetValue("X-Api-Key", out var received)
           || !CryptographicOperations.FixedTimeEquals(
                  Encoding.UTF8.GetBytes(received.ToString()),
                  Encoding.UTF8.GetBytes(expected)))
       {
           return TypedResults.Problem(title: "API key inválida", statusCode: StatusCodes.Status401Unauthorized);
       }

       return await next(context);
   });

app.Run();
```

```text
GET /data                          → 401 ProblemDetails
GET /data  X-Api-Key: otra-cosa    → 401 ProblemDetails   (con la guía original: 200)
GET /data  X-Api-Key: dev-key-123  → 200 "Información sensible"
```

</details>

**Qué observar:**

* `FixedTimeEquals` tarda lo mismo sin importar cuántos caracteres coinciden; una comparación con `==` corta en el primer carácter distinto y, en teoría, permite adivinar la clave midiendo tiempos.
* La clave nunca va en el código ni en el repositorio: en producción, User Secrets, variables de entorno o un almacén de secretos.
* Para APIs con usuarios, la herramienta correcta es `AddAuthentication` + `RequireAuthorization`, no un filtro.

-----

### Ejercicio 3: filtro reutilizable como clase

**Objetivo:** un filtro de logging aplicado a varios endpoints.

**Contexto:** la guía usaba `Console.WriteLine` y afirmaba que eso es "separación total de responsabilidades (Clean Architecture)". Un filtro separa una responsabilidad transversal del handler, lo cual es bueno, pero Clean Architecture trata de capas y dependencias, no de filtros.

**Instrucciones:**

1. `LoggingFilter : IEndpointFilter` con `ILogger<LoggingFilter>` inyectado.
2. Registra método, ruta, código de resultado y duración, aunque el handler lance una excepción.
3. Aplícalo a un grupo.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var products = app.MapGroup("/products").AddEndpointFilter<LoggingFilter>();
products.MapGet("/", () => FakeDb.Products);
products.MapGet("/{id:int}", Results<Ok<Product>, NotFound> (int id) =>
    FakeDb.Products.FirstOrDefault(p => p.Id == id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound());

app.Run();

public sealed class LoggingFilter(ILogger<LoggingFilter> logger) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var start = Stopwatch.GetTimestamp();
        object? result = null;
        try
        {
            result = await next(context);
            return result;
        }
        finally
        {
            var status = (result as IStatusCodeHttpResult)?.StatusCode ?? StatusCodes.Status200OK;
            logger.LogInformation("{Method} {Path} → {Status} en {Elapsed:N1} ms",
                context.HttpContext.Request.Method,
                context.HttpContext.Request.Path,
                result is null ? "excepción" : status,
                Stopwatch.GetElapsedTime(start).TotalMilliseconds);
        }
    }
}

// + Product y FakeDb
```

```text
info: LoggingFilter[0] GET /products/1 → 200 en 0,2 ms
info: LoggingFilter[0] GET /products/9 → 404 en 0,1 ms
```

</details>

**Qué observar:**

* `ILogger` en lugar de `Console.WriteLine`: niveles, mensajes estructurados (`{Method}` queda como campo consultable) y destinos configurables.
* `IStatusCodeHttpResult` es la interfaz que implementan los resultados con código (`Ok<T>`, `NotFound`...). Si el handler devuelve un valor plano (como la lista de `/products/`), no la implementa y será 200.
* El filtro del grupo envuelve a todos los endpoints del grupo: se escribe una vez.

-----

### Ejercicio 4: Typed Results + filtro de autorización

**Objetivo:** combinar un filtro con un handler tipado, sin mentir en el contrato.

**Contexto:** la guía tenía tres problemas:

1. `Results<Ok<Order>, NotFound, Unauthorized>` **no compila**: el tipo se llama `UnauthorizedHttpResult` (CS0246).
2. Aunque compilara, el handler nunca devuelve 401: lo devuelve el filtro. La unión declara algo que el handler no hace.
3. `ContainsKey("Authorization")` no autentica: `Authorization: x` pasa.

**Instrucciones:**

1. El handler declara solo lo que devuelve: `Results<Ok<Order>, NotFound>`.
2. Usa autenticación real con `RequireAuthorization()` (para el ejercicio, el esquema de prueba de abajo; en un proyecto real, JWT).
3. Agrega un filtro con una regla que sí depende de los argumentos: un usuario solo puede ver **sus** pedidos (403 si no).

<details>
<summary>Solución</summary>

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
builder.Services.AddAuthentication("Demo")
    .AddScheme<AuthenticationSchemeOptions, DemoAuthHandler>("Demo", null);
builder.Services.AddAuthorization();

var app = builder.Build();
app.UseStatusCodePages();
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/orders/{id:int}", OrderHandlers.GetById)
   .RequireAuthorization()                       // 401 si no hay usuario autenticado
   .AddEndpointFilter(async (context, next) =>   // 403 si el pedido no es suyo
   {
       var id = context.GetArgument<int>(0);
       var order = FakeDb.Orders.FirstOrDefault(o => o.Id == id);
       var customerId = context.HttpContext.User.FindFirstValue("customer_id");

       if (order is not null && order.CustomerId.ToString() != customerId)
           return TypedResults.Problem(title: "Pedido ajeno", statusCode: StatusCodes.Status403Forbidden);

       return await next(context);
   })
   .ProducesProblem(StatusCodes.Status403Forbidden);   // documenta lo que agrega el filtro

app.Run();

public static class OrderHandlers
{
    public static Results<Ok<Order>, NotFound> GetById(int id) =>
        FakeDb.Orders.FirstOrDefault(o => o.Id == id) is { } order
            ? TypedResults.Ok(order)
            : TypedResults.NotFound();
}

// Esquema de autenticación SOLO para practicar: "Authorization: Demo 10" autentica al cliente 10.
// En producción: AddJwtBearer (paquete Microsoft.AspNetCore.Authentication.JwtBearer).
public sealed class DemoAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var header = Request.Headers.Authorization.ToString();
        if (!header.StartsWith("Demo ") || !int.TryParse(header["Demo ".Length..], out var customerId))
            return Task.FromResult(AuthenticateResult.NoResult());

        var identity = new ClaimsIdentity([new Claim("customer_id", customerId.ToString())], Scheme.Name);
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme.Name)));
    }
}

// + Order y FakeDb
```

```text
GET /orders/1                              → 401 (sin usuario)
GET /orders/1  Authorization: cualquiera   → 401 (con la guía original: pasaba)
GET /orders/1  Authorization: Demo 10      → 200 (el pedido 1 es del cliente 10)
GET /orders/2  Authorization: Demo 10      → 403 "Pedido ajeno"
GET /orders/9  Authorization: Demo 10      → 404
```

</details>

**Qué observar:**

* **Autenticación** (¿quién eres?) es del middleware; **autorización por recurso** (¿este pedido es tuyo?) depende del argumento `id`, y ahí un filtro sí tiene sentido.
* La unión del handler dice la verdad (200 o 404); el 401 lo documenta `RequireAuthorization` y el 403, `ProducesProblem`.
* `GetArgument<int>(0)` depende de que `id` sea el primer parámetro; en un filtro reutilizable conviene buscar por nombre o por tipo.

-----

### Ejercicio 5: filtro de transformación de respuesta

**Objetivo:** entender qué recibe un filtro "después" y cuándo transformar es buena idea.

**Contexto:** en la guía, `/time` devolvía `DateTime.UtcNow` y el filtro lo envolvía en `{ ServerTime, Message }`. Funciona **solo** porque el handler devuelve un valor plano. Si el handler devuelve `TypedResults.Ok(...)`, `result` es el objeto `Ok<DateTime>`, y se serializa como `{ "serverTime": { "value": ..., "statusCode": 200 } }`. Además, OpenAPI sigue documentando un `DateTime`, y un 404 del handler saldría envuelto en un 200.

**Instrucciones:**

1. Escribe un filtro que agregue el header `X-Server-Time` a todas las respuestas del grupo (transformación segura: no toca el cuerpo).
2. Escribe un filtro que convierta los precios de `Ok<Product>` a otra moneda **solo** si el resultado es de ese tipo, y deje pasar todo lo demás sin tocar.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
var app = builder.Build();
app.UseStatusCodePages();

var products = app.MapGroup("/products")
    .AddEndpointFilter(async (context, next) =>
    {
        context.HttpContext.Response.Headers["X-Server-Time"] = DateTime.UtcNow.ToString("O");
        return await next(context);
    });

products.MapGet("/{id:int}", Results<Ok<Product>, NotFound> (int id) =>
        FakeDb.Products.FirstOrDefault(p => p.Id == id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound())
    .AddEndpointFilter(async (context, next) =>
    {
        var result = await next(context);

        var currency = context.HttpContext.Request.Query["currency"].ToString();
        if (currency == "USD" && result is Results<Ok<Product>, NotFound> { Result: Ok<Product> ok })
            return TypedResults.Ok(ok.Value! with { Price = decimal.Round(ok.Value!.Price / 3.75m, 2) });

        return result;   // NotFound, otra moneda o cualquier otro tipo: sin cambios
    });

app.Run();

// + Product y FakeDb
```

```text
GET /products/1                 → 200 { "id": 1, "name": "Keyboard", "price": 150 }   X-Server-Time: 2026-...
GET /products/1?currency=USD    → 200 { "id": 1, "name": "Keyboard", "price": 40 }
GET /products/9?currency=USD    → 404 (no se envolvió en un 200)
```

</details>

**Qué observar:**

* Cuando el handler declara `Results<...>`, lo que recibe el filtro es **la unión**; el resultado concreto está en `.Result`. El patrón `{ Result: Ok<Product> ok }` lo inspecciona sin romper nada.
* El tipo de cambio fijo (3.75) es solo para el ejercicio; en un caso real viene de un servicio.
* Si "centralizar el formato" significa un envoltorio `{ success, data, message }`, vuelve a [ProblemDetails](../01-diseno-de-apis-rest/02-Errores%20con%20ProblemDetails.md): éxito = 2xx + recurso, error = ProblemDetails.

-----

## Retos

### Reto 1: API tipada completa

**Misión:** `GET /products`, `GET /products/{id}` y `POST /products`, todos con Typed Results. El POST responde 201 con `Location`, 400 con errores por campo o 409 si el nombre ya existe.

**Pista:** `Results<Created<Product>, ValidationProblem, Conflict>`.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
var app = builder.Build();
app.UseStatusCodePages();

var products = app.MapGroup("/products");
products.MapGet("/", ProductApi.GetAll);
products.MapGet("/{id:int}", ProductApi.GetById);
products.MapPost("/", ProductApi.Create);

app.Run();

public record CreateProduct(string Name, decimal Price);

public static class ProductApi
{
    public static Ok<List<Product>> GetAll() => TypedResults.Ok(FakeDb.Products);

    public static Results<Ok<Product>, NotFound> GetById(int id) =>
        FakeDb.Products.FirstOrDefault(p => p.Id == id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound();

    public static Results<Created<Product>, ValidationProblem, Conflict> Create(CreateProduct dto)
    {
        var errors = new Dictionary<string, string[]>();
        if (string.IsNullOrWhiteSpace(dto.Name)) errors[nameof(dto.Name)] = ["El nombre es obligatorio."];
        if (dto.Price <= 0) errors[nameof(dto.Price)] = ["El precio debe ser mayor que cero."];
        if (errors.Count > 0) return TypedResults.ValidationProblem(errors);

        if (FakeDb.Products.Any(p => p.Name.Equals(dto.Name.Trim(), StringComparison.OrdinalIgnoreCase)))
            return TypedResults.Conflict();

        var product = new Product(FakeDb.Products.Max(p => p.Id) + 1, dto.Name.Trim(), dto.Price);
        FakeDb.Products.Add(product);
        return TypedResults.Created($"/products/{product.Id}", product);
    }
}

// + Product y FakeDb
```

</details>

### Reto 2: autenticación reutilizable para varios endpoints

**Misión:** aplicar la misma protección a varios endpoints sin repetirla.

**Pista:** la guía pedía "un filtro reutilizable que valide el token". La forma idiomática es un **grupo** con `RequireAuthorization()`: la validación del token la hace el middleware de autenticación, y el grupo la aplica a todos sus endpoints. El filtro queda para reglas por recurso.

<details>
<summary>Solución</summary>

```csharp
// Con el DemoAuthHandler del ejercicio 4 (o AddJwtBearer en un proyecto real):
var api = app.MapGroup("/api")
    .RequireAuthorization()                         // todos los endpoints del grupo exigen usuario
    .AddEndpointFilter<LoggingFilter>();            // y todos se registran

api.MapGet("/orders/{id:int}", OrderHandlers.GetById);
api.MapGet("/products", () => FakeDb.Products);

app.MapGet("/health", () => "ok");                 // fuera del grupo: público
```

Si de verdad necesitas una API key entre servicios, escribe el `ApiKeyFilter` como clase (como en la lección) y agrégalo al grupo con `.AddEndpointFilter<ApiKeyFilter>()`. Un paso mejor: convertirlo en un `AuthenticationHandler` como el `DemoAuthHandler`, para usar `RequireAuthorization()` también con API keys.

</details>

### Reto 3: pipeline completo con medición

**Misión:** combina logging con medición de tiempos, autenticación y Typed Results en un grupo. Mide con `Stopwatch` y advierte en el log las peticiones que tarden más de 200 ms.

**Pista:** `Stopwatch.GetTimestamp()` y `Stopwatch.GetElapsedTime(inicio)` (sin crear objetos `Stopwatch`). Ten en cuenta qué **no** mide un filtro.

<details>
<summary>Solución</summary>

```csharp
public sealed class TimingFilter(ILogger<TimingFilter> logger) : IEndpointFilter
{
    private static readonly TimeSpan Threshold = TimeSpan.FromMilliseconds(200);

    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var start = Stopwatch.GetTimestamp();
        try
        {
            return await next(context);
        }
        finally
        {
            var elapsed = Stopwatch.GetElapsedTime(start);
            var level = elapsed > Threshold ? LogLevel.Warning : LogLevel.Information;
            logger.Log(level, "{Method} {Path} en {Elapsed:N1} ms",
                context.HttpContext.Request.Method, context.HttpContext.Request.Path, elapsed.TotalMilliseconds);
        }
    }
}

// En Program.cs (con autenticación configurada como en el ejercicio 4):
var api = app.MapGroup("/api")
    .RequireAuthorization()
    .AddEndpointFilter<TimingFilter>();

api.MapGet("/products/{id:int}", async Task<Results<Ok<Product>, NotFound>> (int id, CancellationToken ct) =>
{
    await Task.Delay(Random.Shared.Next(50, 400), ct);   // simula la base de datos
    return FakeDb.Products.FirstOrDefault(p => p.Id == id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound();
});
```

```text
info: TimingFilter[0] GET /api/products/1 en 120,4 ms
warn: TimingFilter[0] GET /api/products/1 en 352,8 ms
```

El filtro mide los filtros internos y el handler. **No** mide la autenticación (middleware, antes), el binding ni la serialización de la respuesta (después). Para la duración real de la petición, ASP.NET Core publica la métrica `http.server.request.duration` (visible con `dotnet-counters` u OpenTelemetry).

</details>

-----

## Checkpoint

Antes de seguir, deberías poder responder sin mirar:

* ¿Qué ganas realmente con `TypedResults` frente a `Results`? ¿Y qué **no** ganas?
* ¿Cómo se llama el tipo que devuelve `TypedResults.Unauthorized()`?
* ¿Por qué `ContainsKey("Authorization")` no es autenticación?
* ¿En qué orden corren el filtro de un grupo y dos filtros del endpoint?
* ¿Qué recibe un filtro en `await next(context)` si el handler devuelve `Results<Ok<T>, NotFound>`?
* ¿Qué parte del tiempo de una petición no ve un filtro?
