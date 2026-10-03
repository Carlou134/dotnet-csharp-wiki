# Endpoint filters

## En una frase

Un **endpoint filter** es código que se ejecuta **antes y después** del handler de un endpoint (o de un grupo), con acceso a sus argumentos y a su resultado; sirve para validar, registrar o cortar la petición sin repetir esa lógica en cada handler.

-----

## Antes de empezar

Conviene que ya sepas:

* Endpoints, grupos y Typed Results: [Endpoints y grupos de rutas](01-Endpoints%20y%20grupos%20de%20rutas.md) y [Typed Results](02-Typed%20Results.md).
* ProblemDetails: [Errores con ProblemDetails](../01-diseno-de-apis-rest/02-Errores%20con%20ProblemDetails.md).
* `async`/`await` y `ValueTask`: [Programación asíncrona](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/01-Programacion%20asincrona.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Endpoint filter:** componente que envuelve la ejecución de un handler, con acceso a sus argumentos y a su resultado.
* **`IEndpointFilter`:** interfaz para escribir un filtro como clase reutilizable.
* **`EndpointFilterInvocationContext`:** el contexto del filtro: `HttpContext`, los argumentos del handler (`Arguments`) y `GetArgument<T>(índice)`.
* **`next`:** delegado que ejecuta el siguiente filtro o, al final, el handler.
* **Cortocircuito (*short-circuit*):** devolver una respuesta sin llamar a `next`; el handler no se ejecuta.
* **Middleware:** componente del pipeline que procesa **todas** las peticiones, antes de que se sepa (o se ejecute) el handler.

-----

## El problema

Tres endpoints que crean y modifican productos:

```csharp
static IResult Crear(CrearProducto dto, ILogger<Program> logger, ...)
{
    var inicio = Stopwatch.GetTimestamp();
    if (string.IsNullOrWhiteSpace(dto.Nombre)) return Results.ValidationProblem(...);
    if (dto.Precio <= 0) return Results.ValidationProblem(...);
    // lógica real: 3 líneas
    logger.LogInformation("Crear en {Ms} ms", Stopwatch.GetElapsedTime(inicio).TotalMilliseconds);
    ...
}

static IResult Actualizar(int id, CrearProducto dto, ILogger<Program> logger, ...)
{
    var inicio = Stopwatch.GetTimestamp();
    if (string.IsNullOrWhiteSpace(dto.Nombre)) return Results.ValidationProblem(...);   // copiado
    if (dto.Precio <= 0) return Results.ValidationProblem(...);                          // copiado
    ...
}
```

La validación y la medición se repiten en cada handler. Cuando la regla cambia, hay que cambiarla en todos (y alguno se olvida). Y la lógica real queda escondida entre líneas de "infraestructura".

-----

## Cómo funciona

### 1. Anatomía de un filtro

```csharp
app.MapPost("/api/productos", Crear)
   .AddEndpointFilter(async (context, next) =>
   {
       // 1. ANTES: se puede leer HttpContext y los argumentos del handler
       var dto = context.GetArgument<CrearProducto>(0);

       // 2. CORTOCIRCUITO: devolver sin llamar a next → el handler no se ejecuta
       if (string.IsNullOrWhiteSpace(dto.Nombre))
           return TypedResults.ValidationProblem(new Dictionary<string, string[]>
           {
               ["Nombre"] = ["El nombre es obligatorio."]
           });

       // 3. CONTINUAR: ejecuta el siguiente filtro o el handler
       var resultado = await next(context);

       // 4. DESPUÉS: se puede inspeccionar (o reemplazar) el resultado
       return resultado;
   });
```

* `context.Arguments` contiene los argumentos **ya enlazados** del handler, en orden. `GetArgument<T>(i)` los lee con tipo.
* Lo que devuelve el filtro es lo que se escribe en la respuesta.

### 2. Orden: como capas de cebolla

```text
 petición ──► middleware (UseExceptionHandler, autenticación, autorización...)
                 │
                 ▼  ruteo eligió el endpoint y enlazó los argumentos
        ┌───────────────────── Filtro A (registrado primero; o del grupo) ─────────────────────┐
        │  antes A                                                                  después A │
        │      ┌─────────────────────── Filtro B ───────────────────────┐                     │
        │      │  antes B                                     después B │                     │
        │      │      ┌──────────── handler ────────────┐               │                     │
        │      │      │  Results<Ok<Producto>, ...>      │               │                     │
        │      │      └──────────────────────────────────┘               │                     │
        │      └─────────────────────────────────────────────────────────┘                     │
        └──────────────────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼  el resultado se serializa y se escribe DESPUÉS de los filtros
```

* El primer filtro registrado es el **más externo**: su "antes" corre primero y su "después", último.
* Los filtros de un **grupo** envuelven a los de cada endpoint del grupo.
* Si un filtro cortocircuita, los internos y el handler no se ejecutan.

### 3. Filtros reutilizables: `IEndpointFilter`

```csharp
public sealed class LoggingFilter(ILogger<LoggingFilter> logger) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var inicio = Stopwatch.GetTimestamp();
        try
        {
            return await next(context);
        }
        finally
        {
            var http = context.HttpContext;
            logger.LogInformation("{Metodo} {Ruta} ejecutado en {Ms:N1} ms",
                http.Request.Method, http.Request.Path, Stopwatch.GetElapsedTime(inicio).TotalMilliseconds);
        }
    }
}

grupo.AddEndpointFilter<LoggingFilter>();   // se aplica a todos los endpoints del grupo
```

* Las dependencias llegan por el constructor (`ILogger<T>`, `IConfiguration`, servicios singleton). Para servicios *scoped* (un `DbContext`), resuélvelos dentro de `InvokeAsync` con `context.HttpContext.RequestServices`.
* `ILogger` en lugar de `Console.WriteLine`: niveles, estructura, destinos configurables y sin bloquear la consola.
* `try/finally`: se registra el tiempo aunque el handler lance una excepción.

### 4. Filtro o middleware

| | Middleware | Endpoint filter |
| --- | --- | --- |
| Se aplica a | Todas las peticiones (o un `Map` por rama) | Un endpoint o un grupo |
| Conoce el endpoint | Solo si va después del ruteo, y sin argumentos | Sí, con sus argumentos ya enlazados |
| Ve el resultado | Solo la respuesta ya escrita (bytes) | El objeto resultado, antes de serializar |
| Ideal para | Excepciones, autenticación, CORS, compresión, *rate limiting*, métricas globales | Validación de argumentos, reglas de un grupo de endpoints, auditoría por endpoint |

No compiten: el middleware no es "pesado" ni algo a evitar. Cada uno resuelve un nivel distinto.

### 5. ⚠️ Autenticación: no la reimplementes en un filtro

```csharp
// ❌ Esto NO es autenticación
if (!context.HttpContext.Request.Headers.ContainsKey("Authorization"))
    return TypedResults.Unauthorized();
```

Cualquier petición con `Authorization: lo-que-sea` pasa. No se valida la firma del token, ni su expiración, ni quién es el usuario. Lo correcto es el sistema de autenticación de ASP.NET Core:

```csharp
builder.Services.AddAuthentication().AddJwtBearer();   // paquete Microsoft.AspNetCore.Authentication.JwtBearer
builder.Services.AddAuthorization();

var api = app.MapGroup("/api").RequireAuthorization();  // todo el grupo exige un usuario autenticado
```

El middleware de autenticación valida el token y llena `HttpContext.User`; `RequireAuthorization` exige que exista (y, con políticas, roles o claims). Los filtros sirven después, para reglas como "el usuario solo puede ver **sus** pedidos".

Una **API key** simple entre servicios puede validarse en un filtro para aprender, siempre que se **compare** contra un valor secreto de la configuración (no solo comprobar que el header exista) y con una comparación de tiempo constante. En producción, mejor un `AuthenticationHandler` propio, para integrarse con `RequireAuthorization`.

### 6. Transformar la respuesta: con cuidado

```csharp
app.MapGet("/api/hora", () => TypedResults.Ok(DateTime.UtcNow))
   .AddEndpointFilter(async (context, next) =>
   {
       var resultado = await next(context);
       return TypedResults.Ok(new { ServerTime = resultado, Message = "Procesado" });
   });
```

```json
{ "serverTime": { "value": "2026-10-02T15:00:00Z", "statusCode": 200, ... }, "message": "Procesado" }
```

Cuando el handler devuelve un `IResult` (como `Ok<DateTime>`), `resultado` es **ese objeto**, no el valor: envolverlo serializa sus propiedades internas. Además:

* OpenAPI sigue documentando el tipo original del handler: el contrato publicado **miente**.
* Un 404 o un 400 del handler también quedaría envuelto en un 200.

Si necesitas modificar el resultado, comprueba el tipo (`if (resultado is Ok<DateTime> ok) ...`) y deja pasar el resto sin tocar. Y el envoltorio `{ success, data }` tiene los problemas que viste en [ProblemDetails](../01-diseno-de-apis-rest/02-Errores%20con%20ProblemDetails.md).

### 7. Validación: en .NET 10 ya viene integrada

Escribir un filtro de validación es un buen ejercicio, pero desde **.NET 10** las minimal APIs validan los atributos de DataAnnotations solas:

```csharp
builder.Services.AddValidation();
// Los parámetros con [Required], [Range], [StringLength]... se validan antes del handler
// y, si fallan, se responde 400 con ProblemDetails, sin escribir ningún filtro.
```

En versiones anteriores (o para reglas propias, o con FluentValidation), un filtro genérico es la herramienta habitual (ver el ejemplo completo).

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web`. En `appsettings.Development.json` agrega `"ApiKey": "clave-de-desarrollo-123"` (en producción, va en un almacén de secretos, nunca en el repositorio).

```csharp
using System.Collections.Concurrent;
using System.ComponentModel.DataAnnotations;
using System.Diagnostics;
using System.Security.Cryptography;
using System.Text;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ProductoRepository>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();

var productos = app.MapGroup("/api/productos")
    .WithTags("Productos")
    .AddEndpointFilter<LoggingFilter>();                 // externo: mide todo lo de adentro

productos.MapGet("/{id:int}", ProductosEndpoints.Obtener);

productos.MapPost("/", ProductosEndpoints.Crear)
    .AddEndpointFilter<ApiKeyFilter>()                   // luego la API key
    .AddEndpointFilter<ValidacionFilter<CrearProducto>>(); // y por último la validación

app.Run();

public record Producto(int Id, string Nombre, decimal Precio);

// Record no posicional: Validator lee los atributos de las PROPIEDADES.
public record CrearProducto
{
    [Required, StringLength(100)] public string Nombre { get; init; } = "";
    [Range(0.01, 1_000_000)] public decimal Precio { get; init; }
}

public class ProductoRepository
{
    private readonly ConcurrentDictionary<int, Producto> _datos = new() { [1] = new Producto(1, "Teclado", 150m) };
    private int _ultimoId = 1;

    public Producto? Buscar(int id) => _datos.GetValueOrDefault(id);

    public Producto Agregar(CrearProducto dto)
    {
        var p = new Producto(Interlocked.Increment(ref _ultimoId), dto.Nombre.Trim(), dto.Precio);
        _datos[p.Id] = p;
        return p;
    }
}

public static class ProductosEndpoints
{
    public static Results<Ok<Producto>, NotFound> Obtener(int id, ProductoRepository repo) =>
        repo.Buscar(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound();

    // El handler ya no valida: cuando se ejecuta, el DTO es válido.
    public static Created<Producto> Crear(CrearProducto dto, ProductoRepository repo)
    {
        var producto = repo.Agregar(dto);
        return TypedResults.Created($"/api/productos/{producto.Id}", producto);
    }
}

public sealed class LoggingFilter(ILogger<LoggingFilter> logger) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var inicio = Stopwatch.GetTimestamp();
        try
        {
            return await next(context);
        }
        finally
        {
            var http = context.HttpContext;
            logger.LogInformation("{Metodo} {Ruta} ejecutado en {Ms:N1} ms",
                http.Request.Method, http.Request.Path, Stopwatch.GetElapsedTime(inicio).TotalMilliseconds);
        }
    }
}

public sealed class ApiKeyFilter(IConfiguration configuracion) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var esperada = configuracion["ApiKey"]
            ?? throw new InvalidOperationException("Falta configurar 'ApiKey'.");

        if (!context.HttpContext.Request.Headers.TryGetValue("X-Api-Key", out var recibida)
            || !SonIguales(recibida.ToString(), esperada))
        {
            return TypedResults.Problem(
                title: "API key inválida o ausente",
                statusCode: StatusCodes.Status401Unauthorized);
        }

        return await next(context);
    }

    // Comparación en tiempo constante: no revela cuántos caracteres coinciden.
    private static bool SonIguales(string a, string b) =>
        CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(a), Encoding.UTF8.GetBytes(b));
}

public sealed class ValidacionFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var modelo = context.Arguments.OfType<T>().FirstOrDefault();
        if (modelo is null)
            return TypedResults.Problem(title: "Cuerpo requerido", statusCode: StatusCodes.Status400BadRequest);

        var resultados = new List<ValidationResult>();
        if (Validator.TryValidateObject(modelo, new ValidationContext(modelo), resultados, validateAllProperties: true))
            return await next(context);

        var errores = resultados
            .SelectMany(r => r.MemberNames.DefaultIfEmpty(""), (r, campo) => (Campo: campo, Mensaje: r.ErrorMessage ?? "Valor inválido."))
            .GroupBy(e => e.Campo)
            .ToDictionary(g => g.Key, g => g.Select(e => e.Mensaje).ToArray());

        return TypedResults.ValidationProblem(errores);
    }
}
```

```text
GET  /api/productos/1                                      → 200
POST /api/productos  (sin X-Api-Key)                       → 401 ProblemDetails; la validación y el handler NO corren
POST /api/productos  X-Api-Key: clave-de-desarrollo-123
                     { "nombre": "", "precio": 0 }         → 400 con errors.Nombre y errors.Precio; el handler NO corre
POST /api/productos  X-Api-Key: clave-de-desarrollo-123
                     { "nombre": "Mouse", "precio": 60 }   → 201

Log (una línea por petición, porque LoggingFilter envuelve a todos):
info: LoggingFilter[0]  POST /api/productos ejecutado en 0,4 ms
```

El tiempo que mide `LoggingFilter` es el de los filtros internos y el handler. **No incluye** el ruteo, el binding ni la serialización de la respuesta (que ocurre después de los filtros). Para medir la petición completa, ASP.NET Core ya publica la métrica `http.server.request.duration`, o se usa un middleware.

-----

## Errores comunes

**1. Comprobar que un header existe como "autenticación".**
Qué pasa: cualquiera con un header inventado accede.
Por qué: se confunde presencia con validez.
Arreglo: `AddAuthentication` + `RequireAuthorization`; para API keys, comparar contra un secreto configurado (en tiempo constante) o, mejor, un `AuthenticationHandler`.

**2. Leer la API key pero no compararla.**
Qué pasa: `TryGetValue("x-api-key", out var key)` y nada más; cualquier valor es aceptado.
Por qué: el ejemplo se quedó a medias.
Arreglo: comparar `key` con el valor esperado.

**3. Envolver el resultado sin mirar su tipo.**
Qué pasa: se serializa el `IResult` interno (`value`, `statusCode`...) o un 404 sale como 200.
Por qué: `next` devuelve lo que devolvió el handler, que puede ser un `IResult`.
Arreglo: transforma solo los tipos que conoces y deja pasar el resto.

**4. Orden equivocado de filtros.**
Qué pasa: se valida el cuerpo antes de comprobar la API key, o el log no mide la validación.
Por qué: el primero registrado es el más externo.
Arreglo: de afuera hacia adentro: observabilidad → seguridad → validación → handler.

**5. Índices fijos en `GetArgument<T>(0)`.**
Qué pasa: si alguien reordena los parámetros del handler, `InvalidCastException`.
Por qué: el índice depende de la firma.
Arreglo: en filtros reutilizables, busca por tipo (`context.Arguments.OfType<T>()`), o usa una *filter factory* que inspeccione la firma.

**6. Atributos de validación en un record posicional con `Validator`.**
Qué pasa: `Validator.TryValidateObject` no encuentra errores en `record CrearProducto([Required] string Nombre)`.
Por qué: el atributo queda en el **parámetro** del constructor, y `Validator` lee las **propiedades**.
Arreglo: record con propiedades (`{ get; init; }`) o `[property: Required]`. (La validación de MVC y la de .NET 10 sí leen los parámetros).

**7. Servicios *scoped* en el constructor del filtro.**
Qué pasa: errores de alcance o un `DbContext` compartido entre peticiones.
Por qué: el filtro puede crearse una vez y reutilizarse.
Arreglo: `context.HttpContext.RequestServices.GetRequiredService<T>()` dentro de `InvokeAsync`.

-----

## Según la versión de .NET

* **.NET 7:** endpoint filters (`AddEndpointFilter`, `IEndpointFilter`, `AddEndpointFilterFactory`) y filtros en `MapGroup`.
* **.NET 8:** filtros compatibles con el *Request Delegate Generator* y Native AOT; métrica `http.server.request.duration`.
* **.NET 10:** validación integrada (`AddValidation()`), que reemplaza al filtro de validación típico.

-----

## Cuándo sí y cuándo no

**Usa un endpoint filter cuando:**

* La lógica depende de los **argumentos** del handler (validar el DTO, comprobar que el `id` pertenece al usuario).
* Se aplica a un endpoint o a un grupo, no a toda la aplicación.
* Quieres auditoría o métricas por endpoint.

**Usa otra cosa cuando:**

* Autenticación y autorización → `AddAuthentication` / `RequireAuthorization`.
* Manejo de excepciones global → `UseExceptionHandler` + `IExceptionHandler`.
* Cache de respuestas → `AddOutputCache` / `CacheOutput()`.
* *Rate limiting* → `AddRateLimiter` / `RequireRateLimiting()`.
* La regla es del negocio → el dominio, no un filtro HTTP.

-----

## Resumen en 5 líneas

1. Un endpoint filter envuelve al handler: antes, cortocircuito o `next`, después.
2. Tiene acceso a `HttpContext`, a los argumentos enlazados y al resultado.
3. Se aplica por endpoint o por grupo; el primero registrado es el más externo.
4. Sirve para validación, auditoría y reglas por endpoint; no para reimplementar autenticación.
5. Transforma resultados solo si conoces su tipo; en .NET 10, la validación de DataAnnotations ya viene integrada.

-----

## Para profundizar

<details>
<summary>Filter factories: decidir al construir el endpoint</summary>

```csharp
grupo.AddEndpointFilterFactory((factoryContext, next) =>
{
    // Se ejecuta UNA vez por endpoint, al iniciar: inspecciona la firma del handler.
    var indice = Array.FindIndex(factoryContext.MethodInfo.GetParameters(),
                                 p => p.ParameterType == typeof(CrearProducto));
    if (indice < 0) return next;   // este endpoint no recibe CrearProducto: no se agrega el filtro

    return async invocationContext =>
    {
        var dto = invocationContext.GetArgument<CrearProducto>(indice);
        // validar...
        return await next(invocationContext);
    };
});
```

El costo de buscar el parámetro se paga una vez, no en cada petición, y el filtro solo se aplica donde tiene sentido.

</details>

<details>
<summary>Probar un filtro</summary>

Los filtros como clase se pueden probar creando un `DefaultHttpContext`, un `DefaultEndpointFilterInvocationContext(httpContext, argumentos...)` y un `next` falso (`_ => ValueTask.FromResult<object?>("ok")`), y verificando si se llamó a `next` o qué resultado se devolvió.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un endpoint filter es código que se ejecuta antes y después de un endpoint de minimal API. Puede leer los argumentos, cortar la petición devolviendo una respuesta (por ejemplo un 400 si los datos no son válidos) o modificar el resultado. Se agrega con `AddEndpointFilter` a un endpoint o a un grupo, y se puede escribir como clase con `IEndpointFilter` para reutilizarlo.

### Respuesta ampliada (semi-senior)

Los endpoint filters son el equivalente de los action filters de MVC para minimal APIs: envuelven al handler con acceso a los argumentos enlazados y al resultado, en orden de cebolla. Los uso para validación de DTOs (o la validación integrada de .NET 10), auditoría y reglas a nivel de endpoint o grupo, aplicados con `MapGroup`. No los uso para autenticación, excepciones, cache o rate limiting, que tienen middleware dedicado. Cuido que los filtros reutilizables no dependan de índices de argumentos (o uso filter factories), que resuelvan servicios scoped desde `RequestServices` y que no envuelvan resultados sin conocer su tipo, porque rompen el contrato de OpenAPI. Para medir la duración completa de la petición uso las métricas de ASP.NET Core, porque el filtro no incluye la serialización.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre un endpoint filter y un middleware?**
El middleware actúa sobre todas las peticiones y ve bytes; el filtro actúa sobre un endpoint o grupo y ve argumentos y resultados como objetos.

**2. ¿En qué orden se ejecutan varios filtros?**
El primero registrado es el más externo; los del grupo envuelven a los del endpoint.

**3. ¿Por qué no validar un token JWT en un filtro?**
Porque el sistema de autenticación ya lo hace bien (firma, expiración, emisor, claims) y se integra con la autorización y con OpenAPI.

-----

## Práctica

**Ejercicio 1.** Escribe un filtro `RequiereHeaderTenantFilter` que exija el header `X-Tenant-Id` con un `Guid` válido y lo guarde en `HttpContext.Items["TenantId"]` para el handler. Si falta o es inválido, 400 ProblemDetails.

<details>
<summary>Solución</summary>

```csharp
public sealed class RequiereHeaderTenantFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var http = context.HttpContext;
        if (!Guid.TryParse(http.Request.Headers["X-Tenant-Id"], out var tenantId))
            return TypedResults.Problem(
                title: "Tenant inválido",
                detail: "El header X-Tenant-Id debe contener un GUID válido.",
                statusCode: StatusCodes.Status400BadRequest);

        http.Items["TenantId"] = tenantId;
        return await next(context);
    }
}

// Uso: app.MapGroup("/api").AddEndpointFilter<RequiereHeaderTenantFilter>();
```

`Guid.TryParse` acepta el `StringValues` del header porque se convierte implícitamente a `string` (null si no existe, y `TryParse(null)` devuelve `false`).

</details>

**Ejercicio 2.** Dados estos registros, ¿en qué orden se escriben los mensajes?

```csharp
var g = app.MapGroup("/x").AddEndpointFilter(async (c, n) => { Log("G antes"); var r = await n(c); Log("G después"); return r; });
g.MapGet("/", () => { Log("handler"); return "ok"; })
 .AddEndpointFilter(async (c, n) => { Log("A antes"); var r = await n(c); Log("A después"); return r; })
 .AddEndpointFilter(async (c, n) => { Log("B antes"); var r = await n(c); Log("B después"); return r; });
```

<details>
<summary>Solución</summary>

```text
G antes
A antes
B antes
handler
B después
A después
G después
```

El filtro del grupo es el más externo; entre los del endpoint, el primero registrado (A) envuelve al segundo (B).

</details>

-----

## Siguiente lección

Terminaste el módulo. Para practicar con los ejercicios de clase: [Ejercicios de minimal APIs](Ejercicios.md). Después continúa con [Seguridad de APIs](../03-seguridad-de-apis/README.md) o vuelve al [índice del módulo](README.md).
