# Endpoints y grupos de rutas

## En una frase

Las **minimal APIs** definen cada endpoint con una línea (`app.MapGet(ruta, handler)`) en lugar de una clase controlador, y los **grupos de rutas** (`MapGroup`) reúnen endpoints que comparten prefijo, filtros, autorización y metadatos.

-----

## Antes de empezar

Conviene que ya sepas:

* Recursos, métodos HTTP y códigos de estado: [Principios REST y códigos de estado](../01-diseno-de-apis-rest/01-Principios%20REST%20y%20codigos%20de%20estado.md).
* Lambdas y delegados: [Delegados](../../01-csharp-core-and-runtime/09-delegados-y-eventos/01-Delegados.md).
* Métodos de extensión: [Sintaxis moderna de C#](../../01-csharp-core-and-runtime/11-codigo-limpio/03-Sintaxis%20moderna%20de%20CSharp.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Minimal API:** forma de definir endpoints HTTP con métodos `Map*` y funciones, sin controladores.
* **Handler (manejador):** la función (lambda o método) que atiende la petición de un endpoint.
* **Binding (enlace de parámetros):** el proceso que llena los parámetros del handler desde la ruta, la query string, los headers, el cuerpo o los servicios.
* **Grupo de rutas (`MapGroup`):** conjunto de endpoints con un prefijo común y configuración compartida.
* **Metadatos del endpoint:** información asociada a un endpoint (nombre, tags, autorización, tipos de respuesta) que usan el ruteo, la seguridad y OpenAPI.

-----

## El problema

Con controladores, un endpoint trivial necesita bastante estructura:

```csharp
[ApiController]
[Route("api/productos")]
public class ProductosController(IProductoRepository repo) : ControllerBase
{
    [HttpGet("{id:int}")]
    public ActionResult<Producto> Obtener(int id) => repo.Buscar(id) is { } p ? p : NotFound();
}
```

Para un microservicio con cinco endpoints, la clase, la herencia, los atributos y las convenciones de MVC son más ceremonia que lógica. Y el pipeline de MVC (filtros, model binding, formatters) agrega un costo que no siempre necesitas.

Pero si escribes todo en `Program.cs` sin orden, el archivo crece a 800 líneas con rutas repetidas (`/api/productos`, `/api/productos/{id}`...) y la misma configuración copiada en cada endpoint.

-----

## Cómo funciona

### 1. Un endpoint = método HTTP + ruta + handler

```csharp
var app = WebApplication.Create(args);

app.MapGet("/api/saludo/{nombre}", (string nombre) => $"Hola, {nombre}");
app.MapPost("/api/productos", (CrearProducto dto) => ...);
app.MapPut("/api/productos/{id:int}", (int id, CrearProducto dto) => ...);
app.MapDelete("/api/productos/{id:int}", (int id) => ...);

app.Run();
```

Lo que devuelve el handler se convierte en la respuesta:

| El handler devuelve | Respuesta |
| --- | --- |
| `string` | 200, `text/plain` |
| Un objeto (`Producto`, record, anónimo) | 200, JSON |
| `IResult` (`Results.NotFound()`, `TypedResults.Ok(x)`) | Lo que indique el resultado (código, headers, cuerpo) |
| `Task<T>` / `ValueTask<T>` | Lo mismo, después de esperar |

### 2. De dónde salen los parámetros (binding)

```csharp
app.MapGet("/api/productos/{id:int}",
    (int id,                              // ruta: coincide con {id}
     string? moneda,                      // query string: ?moneda=PEN
     [FromHeader(Name = "X-Tenant")] string tenant,   // header
     IProductoRepository repo,            // servicio registrado en DI
     HttpContext http,                    // tipos especiales
     CancellationToken ct) => ...);       // se cancela si el cliente se desconecta

app.MapPost("/api/productos", (CrearProducto dto) => ...);   // cuerpo JSON (tipo complejo en POST/PUT)
```

Las reglas por defecto:

* Si el nombre coincide con un parámetro de la ruta → **ruta**.
* Si es un tipo registrado en DI → **servicio**.
* Si es un tipo simple (`int`, `string`, `Guid`...) → **query string**.
* Si es un tipo complejo → **cuerpo JSON** (como máximo uno por endpoint).
* `HttpContext`, `HttpRequest`, `CancellationToken`, `ClaimsPrincipal` → se inyectan solos.

Los atributos `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromServices]` y `[AsParameters]` lo hacen explícito.

### 3. Restricciones de ruta: `{id:int}`

```text
 "/api/productos/{id}"      + int id   →  /api/productos/abc  → 400 Bad Request (falló el binding)
 "/api/productos/{id:int}"  + int id   →  /api/productos/abc  → 404 Not Found   (la ruta no coincide)
```

Con `{id:int}`, la ruta solo coincide con enteros y otro endpoint (por ejemplo `/api/productos/destacados`) puede convivir sin conflictos.

### 4. Grupos de rutas: `MapGroup`

```csharp
var productos = app.MapGroup("/api/productos")
    .WithTags("Productos");                 // agrupa en OpenAPI

productos.MapGet("/", ListarProductos);
productos.MapGet("/{id:int}", ObtenerProducto).WithName("ObtenerProducto");
productos.MapPost("/", CrearProducto);
productos.MapDelete("/{id:int}", EliminarProducto);
```

```text
 MapGroup("/api/productos")  ── .WithTags · .RequireAuthorization · .AddEndpointFilter (se heredan)
      │
      ├── GET    /          → ListarProductos
      ├── GET    /{id:int}  → ObtenerProducto
      ├── POST   /          → CrearProducto
      └── DELETE /{id:int}  → EliminarProducto
```

Todo lo que configuras en el grupo (tags, autorización, filtros, versionado) se aplica a cada endpoint del grupo. Los grupos pueden anidarse: `app.MapGroup("/api").MapGroup("/productos")`.

### 5. Organizar: un archivo por recurso con un método de extensión

`Program.cs` queda como "índice":

```csharp
app.MapProductos();
app.MapPedidos();
```

Y cada recurso vive en su archivo:

```csharp
public static class ProductosEndpoints
{
    public static IEndpointRouteBuilder MapProductos(this IEndpointRouteBuilder app)
    {
        var grupo = app.MapGroup("/api/productos").WithTags("Productos");
        grupo.MapGet("/", Listar);
        grupo.MapGet("/{id:int}", Obtener);
        return app;
    }

    static IEnumerable<Producto> Listar(IProductoRepository repo) => repo.Listar();

    static IResult Obtener(int id, IProductoRepository repo) =>
        repo.Buscar(id) is { } p ? Results.Ok(p) : Results.NotFound();
}
```

Usar **métodos con nombre** en lugar de lambdas largas tiene otra ventaja: se pueden probar con pruebas unitarias llamándolos directamente.

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web`:

```csharp
using System.Collections.Concurrent;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ProductoRepository>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapProductos();
app.Run();

public record Producto(int Id, string Nombre, decimal Precio);
public record CrearProducto(string Nombre, decimal Precio);

public class ProductoRepository
{
    private readonly ConcurrentDictionary<int, Producto> _datos = new()
    {
        [1] = new Producto(1, "Teclado", 150m),
        [2] = new Producto(2, "Mouse", 60m),
    };
    private int _ultimoId = 2;

    public IEnumerable<Producto> Listar(decimal? precioMaximo) =>
        _datos.Values.Where(p => precioMaximo is null || p.Precio <= precioMaximo).OrderBy(p => p.Id);

    public Producto? Buscar(int id) => _datos.GetValueOrDefault(id);

    public Producto Agregar(CrearProducto dto)
    {
        var producto = new Producto(Interlocked.Increment(ref _ultimoId), dto.Nombre.Trim(), dto.Precio);
        _datos[producto.Id] = producto;
        return producto;
    }

    public bool Eliminar(int id) => _datos.TryRemove(id, out _);
}

public static class ProductosEndpoints
{
    public static IEndpointRouteBuilder MapProductos(this IEndpointRouteBuilder app)
    {
        var grupo = app.MapGroup("/api/productos").WithTags("Productos");

        grupo.MapGet("/", Listar);
        grupo.MapGet("/{id:int}", Obtener).WithName("ObtenerProducto");
        grupo.MapPost("/", Crear);
        grupo.MapDelete("/{id:int}", Eliminar);

        return app;
    }

    static IEnumerable<Producto> Listar(decimal? precioMaximo, ProductoRepository repo) =>
        repo.Listar(precioMaximo);

    static IResult Obtener(int id, ProductoRepository repo) =>
        repo.Buscar(id) is { } p ? Results.Ok(p) : Results.NotFound();

    static IResult Crear(CrearProducto dto, ProductoRepository repo)
    {
        if (string.IsNullOrWhiteSpace(dto.Nombre) || dto.Precio <= 0)
            return Results.ValidationProblem(new Dictionary<string, string[]>
            {
                ["producto"] = ["El nombre es obligatorio y el precio debe ser mayor que cero."]
            });

        var producto = repo.Agregar(dto);
        return Results.CreatedAtRoute("ObtenerProducto", new { id = producto.Id }, producto);
    }

    static IResult Eliminar(int id, ProductoRepository repo) =>
        repo.Eliminar(id) ? Results.NoContent() : Results.NotFound();
}
```

```text
GET    /api/productos                    → 200 [ teclado, mouse ]
GET    /api/productos?precioMaximo=100   → 200 [ mouse ]
GET    /api/productos/1                  → 200 { "id": 1, "nombre": "Teclado", "precio": 150 }
GET    /api/productos/9                  → 404 ProblemDetails
GET    /api/productos/abc                → 404 (la restricción :int no coincide)
POST   /api/productos  { "nombre": "Monitor", "precio": 800 }
                                         → 201 Location: http://localhost:5000/api/productos/3
POST   /api/productos  { "nombre": "", "precio": 0 }
                                         → 400 ValidationProblem
DELETE /api/productos/2                  → 204
```

Fíjate en que los handlers devuelven `IResult` y funcionan, pero la firma no dice **qué** pueden devolver: para OpenAPI, `Obtener` es "algo". Eso lo resuelven los [Typed Results](02-Typed%20Results.md).

-----

## Errores comunes

**1. Todo en `Program.cs`.**
Qué pasa: un archivo enorme con rutas y lógica mezcladas.
Por qué: los ejemplos de documentación caben en un archivo y se copian tal cual.
Arreglo: un método de extensión `MapX` por recurso, en su propio archivo, con un `MapGroup`.

**2. Dos parámetros complejos sin atributo.**
Qué pasa: `InvalidOperationException` al iniciar: no se puede inferir de dónde viene cada uno.
Por qué: solo puede haber un cuerpo por petición.
Arreglo: un único DTO en el cuerpo; lo demás, de la ruta, la query string o `[AsParameters]`.

**3. Olvidar `{id:int}`.**
Qué pasa: `/api/productos/abc` responde 400 en lugar de 404, y una ruta literal como `/api/productos/destacados` puede chocar.
Por qué: sin restricción, `{id}` acepta cualquier texto.
Arreglo: restricciones de tipo en la ruta (`:int`, `:guid`, `:min(1)`).

**4. Lambdas de 50 líneas.**
Qué pasa: endpoints imposibles de probar y de leer.
Por qué: la lambda en línea es lo más rápido de escribir.
Arreglo: métodos con nombre (estáticos) que reciben sus dependencias como parámetros; la lógica del negocio, en servicios o en el dominio.

**5. Usar `Results.Ok(x)` para crear recursos.**
Qué pasa: 200 sin `Location`.
Por qué: el mismo error que en controladores.
Arreglo: `Results.CreatedAtRoute("Nombre", new { id }, x)` o `Results.Created($"/api/productos/{id}", x)`.

-----

## Según la versión de .NET

* **.NET 6:** minimal APIs (`MapGet`, `MapPost`...), `WebApplication`.
* **.NET 7:** `MapGroup`, `TypedResults`, endpoint filters, `[AsParameters]`.
* **.NET 8:** binding de formularios, *Request Delegate Generator* (compatible con Native AOT), `[FromKeyedServices]`.
* **.NET 9:** `Microsoft.AspNetCore.OpenApi` integrado en las plantillas (`AddOpenApi`, `MapOpenApi`).
* **.NET 10:** validación integrada para minimal APIs (`builder.Services.AddValidation()`), OpenAPI 3.1 por defecto, `TypedResults.ServerSentEvents`.

-----

## Cuándo sí y cuándo no

**Minimal APIs encajan bien cuando:**

* Microservicios o APIs pequeñas y medianas.
* Quieres el menor overhead posible o compilar con Native AOT.
* Prefieres organizar por recurso o por *feature* (*vertical slices*) en lugar de por controladores.

**Controladores siguen siendo razonables cuando:**

* El proyecto ya usa MVC y sus filtros, convenciones y librerías.
* El equipo está más cómodo con su estructura.

Ambas opciones son soportadas en .NET 10; la diferencia de rendimiento existe, pero en la mayoría de las APIs el tiempo se va en la base de datos y la red, no en el framework.

-----

## Resumen en 5 líneas

1. Un endpoint es `app.MapVerbo(ruta, handler)`; lo que devuelve el handler es la respuesta.
2. Los parámetros se enlazan solos: ruta, query string, cuerpo, servicios y tipos especiales.
3. Usa restricciones de ruta (`{id:int}`) para que la ruta solo coincida con valores válidos.
4. `MapGroup` comparte prefijo y configuración (tags, filtros, autorización) entre endpoints.
5. Un método de extensión `MapX` por recurso y handlers como métodos con nombre: ordenado y probable.

-----

## Para profundizar

<details>
<summary>[AsParameters]: agrupar parámetros en un record</summary>

```csharp
public record FiltroProductos(decimal? PrecioMaximo, int Page = 1, int PageSize = 20);

grupo.MapGet("/", ([AsParameters] FiltroProductos filtro, ProductoRepository repo) => ...);
// GET /api/productos?precioMaximo=100&page=2
```

Cada propiedad del record se enlaza como si fuera un parámetro suelto. Útil cuando un endpoint tiene muchos filtros.

</details>

<details>
<summary>¿Por qué minimal APIs son más livianas?</summary>

Los controladores pasan por el pipeline de MVC: descubrimiento de controladores, model binding con proveedores, filtros de acción y de resultado, formatters. Las minimal APIs generan un `RequestDelegate` específico por endpoint (en tiempo de ejecución, o en compilación con el *Request Delegate Generator*), con el binding resuelto de antemano. Eso reduce asignaciones y permite Native AOT. Aun así, mide antes de migrar por rendimiento: rara vez el framework es el cuello de botella.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las minimal APIs son una forma de crear endpoints en ASP.NET Core sin controladores: con `app.MapGet("/ruta", handler)` defines la ruta y la función que responde. Los parámetros se llenan solos desde la ruta, la query string, el cuerpo o la inyección de dependencias. Con `MapGroup` agrupas endpoints que comparten prefijo y configuración.

### Respuesta ampliada (semi-senior)

Uso minimal APIs organizadas por recurso: un método de extensión `MapX` con un `MapGroup` que aplica tags, autorización y filtros comunes, y handlers como métodos estáticos con nombre, que reciben sus dependencias por parámetro y se prueban unitariamente. Conozco las reglas de binding y las hago explícitas cuando hay ambigüedad (`[FromHeader]`, `[AsParameters]`). Agrego restricciones de ruta para separar 400 de 404. Son más livianas que MVC y compatibles con Native AOT, pero la elección frente a controladores depende más del equipo y del proyecto que del rendimiento.

### Preguntas frecuentes de seguimiento

**1. ¿Qué ventajas tienen las minimal APIs frente a los controladores?**
Menos ceremonia, menos overhead, compatibilidad con Native AOT y organización flexible por feature.

**2. ¿Cómo evitas que `Program.cs` crezca sin control?**
Con un método de extensión por recurso y grupos de rutas.

**3. ¿Cómo se enlaza un parámetro complejo en un GET?**
Por defecto un tipo complejo se lee del cuerpo; en un GET se usa `[AsParameters]` para leerlo de la query string.

-----

## Práctica

**Ejercicio 1.** Crea `MapClientes` con un grupo `/api/clientes` que tenga `GET /` (con filtro opcional `?nombre=`), `GET /{id:guid}` y `DELETE /{id:guid}`, usando un diccionario en memoria.

<details>
<summary>Solución</summary>

```csharp
public record Cliente(Guid Id, string Nombre);

public static class ClientesEndpoints
{
    private static readonly Dictionary<Guid, Cliente> Clientes = new();

    public static IEndpointRouteBuilder MapClientes(this IEndpointRouteBuilder app)
    {
        var grupo = app.MapGroup("/api/clientes").WithTags("Clientes");
        grupo.MapGet("/", Listar);
        grupo.MapGet("/{id:guid}", Obtener);
        grupo.MapDelete("/{id:guid}", Eliminar);
        return app;
    }

    static IEnumerable<Cliente> Listar(string? nombre) =>
        Clientes.Values.Where(c => nombre is null || c.Nombre.Contains(nombre, StringComparison.OrdinalIgnoreCase));

    static IResult Obtener(Guid id) =>
        Clientes.TryGetValue(id, out var c) ? Results.Ok(c) : Results.NotFound();

    static IResult Eliminar(Guid id) =>
        Clientes.Remove(id) ? Results.NoContent() : Results.NotFound();
}
```

(El `Dictionary` estático no es seguro con peticiones concurrentes; en un caso real va un repositorio registrado en DI.)

</details>

**Ejercicio 2.** ¿De dónde se enlaza cada parámetro?

```csharp
app.MapPut("/api/pedidos/{id:int}",
    (int id, int version, ActualizarPedido dto, ILogger<Program> logger, CancellationToken ct) => ...);
```

<details>
<summary>Solución</summary>

* `id` → ruta (coincide con `{id}`).
* `version` → query string (`?version=3`): tipo simple que no está en la ruta.
* `dto` → cuerpo JSON (tipo complejo).
* `logger` → servicio de DI.
* `ct` → token de cancelación de la petición (`HttpContext.RequestAborted`).

</details>

-----

## Siguiente lección

[Typed Results](02-Typed%20Results.md)
