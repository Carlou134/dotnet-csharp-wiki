# Typed Results

## En una frase

Los **Typed Results** (`TypedResults.Ok(x)`, `TypedResults.NotFound()`) son resultados con un tipo concreto, y `Results<T1, T2, ...>` declara en la firma del endpoint **todas** las respuestas posibles; así el compilador impide devolver algo no declarado, OpenAPI documenta las respuestas solo y las pruebas pueden verificar el tipo exacto.

-----

## Antes de empezar

Conviene que ya sepas:

* Endpoints y grupos: [Endpoints y grupos de rutas](01-Endpoints%20y%20grupos%20de%20rutas.md).
* Códigos de estado y ProblemDetails: [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md).
* Genéricos e interfaces: [Genéricos](../../01-csharp-core-and-runtime/05-tipos-avanzados/04-Genericos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`IResult`:** interfaz de todo resultado HTTP en minimal APIs; sabe escribirse en la respuesta.
* **`Results` (clase estática):** fábrica de resultados que devuelve `IResult`.
* **`TypedResults` (clase estática):** fábrica de los mismos resultados, pero devolviendo el **tipo concreto** (`Ok<T>`, `NotFound`...).
* **`Results<T1, ..., T6>` (unión de resultados):** tipo que representa "una de estas respuestas".
* **OpenAPI:** especificación que describe una API HTTP (rutas, parámetros, respuestas) para documentación y generación de clientes.

-----

## El problema

```csharp
app.MapGet("/api/productos/{id:int}", (int id, ProductoRepository repo) =>
{
    var producto = repo.Buscar(id);
    return producto is not null ? Results.Ok(producto) : Results.NotFound();
});
```

Funciona. Pero:

1. **La firma no dice nada:** el handler devuelve `IResult`. Para OpenAPI, este endpoint devuelve... algo. El frontend que genera su cliente desde OpenAPI no sabe que puede recibir un `Producto` o un 404.
2. **Nada impide un error de contrato:** mañana alguien agrega `return Results.BadRequest("...")` y la documentación no se entera.
3. **Probarlo es incómodo:** en una prueba unitaria recibes un `IResult` y tienes que adivinar el tipo real para verificar el código de estado y el cuerpo.

Para documentarlo hay que agregar metadatos a mano, que se desincronizan con el código:

```csharp
.Produces<Producto>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);
```

-----

## Cómo funciona

### 1. `Results` frente a `TypedResults`

```csharp
IResult a = Results.Ok(producto);           // tipo estático: IResult
Ok<Producto> b = TypedResults.Ok(producto); // tipo estático: Ok<Producto>
```

Los dos crean **el mismo objeto** en tiempo de ejecución. La diferencia es lo que **sabe el compilador**: con `TypedResults`, el tipo concreto queda en la firma, y ese tipo implementa `IEndpointMetadataProvider`, que informa a OpenAPI el código y el cuerpo de la respuesta.

| Fábrica | Devuelve | Tipo concreto |
| --- | --- | --- |
| `TypedResults.Ok(x)` | 200 + JSON | `Ok<T>` |
| `TypedResults.Created(uri, x)` | 201 + `Location` | `Created<T>` |
| `TypedResults.CreatedAtRoute(x, "Nombre", new { id })` | 201 + `Location` generado | `CreatedAtRoute<T>` |
| `TypedResults.NoContent()` | 204 | `NoContent` |
| `TypedResults.BadRequest()` | 400 | `BadRequest` |
| `TypedResults.ValidationProblem(errores)` | 400 + ProblemDetails con `errors` | `ValidationProblem` |
| `TypedResults.Unauthorized()` | 401 | **`UnauthorizedHttpResult`** |
| `TypedResults.Forbid()` | 403 (vía autenticación) | **`ForbidHttpResult`** |
| `TypedResults.NotFound()` | 404 | `NotFound` |
| `TypedResults.Conflict()` | 409 | `Conflict` |
| `TypedResults.Problem(...)` | ProblemDetails con el código indicado | **`ProblemHttpResult`** |

⚠️ Algunos tipos terminan en `HttpResult`: `Results<Ok<T>, Unauthorized>` **no compila** (CS0246: no existe el tipo `Unauthorized`); es `UnauthorizedHttpResult`.

### 2. `Results<...>`: el contrato en la firma

```csharp
app.MapGet("/api/productos/{id:int}",
    Results<Ok<Producto>, NotFound> (int id, ProductoRepository repo) =>
        repo.Buscar(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound());
```

* El tipo de retorno de la lambda (`Results<Ok<Producto>, NotFound>`) es **la lista de respuestas posibles**.
* Cada `TypedResults.X()` se convierte implícitamente a la unión **si está en la lista**.
* Si agregas `return TypedResults.BadRequest();` sin sumarlo a la unión: **error CS0029**, no compila. El contrato no se puede romper por accidente.
* Admite de 2 a 6 tipos: `Results<T1, T2>` ... `Results<T1, T2, T3, T4, T5, T6>`.

```text
             ┌──────────────── Results<Ok<Producto>, NotFound, ValidationProblem> ────────────────┐
  handler ──►│  Ok<Producto>      → 200  application/json   Producto                              │──► OpenAPI
             │  NotFound          → 404                                                            │    (sin anotar
             │  ValidationProblem → 400  application/problem+json  HttpValidationProblemDetails    │     nada a mano)
             └─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3. Async

```csharp
static async Task<Results<Ok<Producto>, NotFound>> Obtener(int id, IProductoRepository repo, CancellationToken ct)
{
    var producto = await repo.BuscarAsync(id, ct);
    return producto is null ? TypedResults.NotFound() : TypedResults.Ok(producto);
}
```

### 4. Pruebas unitarias: verificar el tipo exacto

Como el handler es un método con nombre que devuelve un tipo concreto, la prueba no necesita un servidor HTTP:

```csharp
[Fact]
public void Obtener_devuelve_404_si_no_existe()
{
    var repo = new ProductoRepository();

    var resultado = ProductosEndpoints.Obtener(999, repo);

    Assert.IsType<NotFound>(resultado.Result);
}

[Fact]
public void Obtener_devuelve_el_producto()
{
    var resultado = ProductosEndpoints.Obtener(1, new ProductoRepository());

    var ok = Assert.IsType<Ok<Producto>>(resultado.Result);
    Assert.Equal("Teclado", ok.Value!.Nombre);
}
```

`Results<...>.Result` expone el resultado concreto que se eligió.

### 5. ¿Y el rendimiento?

Es habitual leer que los Typed Results "son más rápidos" o "evitan boxing". En la práctica:

* `Results.Ok(x)` y `TypedResults.Ok(x)` crean **el mismo objeto**; los resultados son clases, así que no hay boxing en ningún caso.
* La serialización del cuerpo es la misma.

Los beneficios reales son **el contrato verificado por el compilador, la documentación OpenAPI automática y las pruebas**. El rendimiento de las minimal APIs viene de otro lado (el `RequestDelegate` generado por endpoint, el *Request Delegate Generator* y Native AOT), no de elegir `TypedResults`.

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web` con el paquete `Microsoft.AspNetCore.OpenApi` (`dotnet add package Microsoft.AspNetCore.OpenApi`):

```csharp
using System.Collections.Concurrent;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ProductoRepository>();
builder.Services.AddProblemDetails();
builder.Services.AddOpenApi();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapOpenApi();                 // documento en /openapi/v1.json
app.MapProductos();
app.Run();

public record Producto(int Id, string Nombre, decimal Precio, int Stock);
public record CrearProducto(string Nombre, decimal Precio, int Stock);

public class ProductoRepository
{
    private readonly ConcurrentDictionary<int, Producto> _datos = new()
    {
        [1] = new Producto(1, "Teclado", 150m, 10),
        [2] = new Producto(2, "Mouse", 60m, 0),
    };
    private int _ultimoId = 2;

    public Producto? Buscar(int id) => _datos.GetValueOrDefault(id);
    public bool ExisteNombre(string nombre) =>
        _datos.Values.Any(p => p.Nombre.Equals(nombre.Trim(), StringComparison.OrdinalIgnoreCase));

    public Producto Agregar(CrearProducto dto)
    {
        var p = new Producto(Interlocked.Increment(ref _ultimoId), dto.Nombre.Trim(), dto.Precio, dto.Stock);
        _datos[p.Id] = p;
        return p;
    }

    public bool Eliminar(int id) => _datos.TryRemove(id, out _);
}

public static class ProductosEndpoints
{
    public static IEndpointRouteBuilder MapProductos(this IEndpointRouteBuilder app)
    {
        var grupo = app.MapGroup("/api/productos").WithTags("Productos");
        grupo.MapGet("/{id:int}", Obtener).WithName("ObtenerProducto");
        grupo.MapPost("/", Crear);
        grupo.MapDelete("/{id:int}", Eliminar);
        return app;
    }

    public static Results<Ok<Producto>, NotFound> Obtener(int id, ProductoRepository repo) =>
        repo.Buscar(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound();

    public static Results<CreatedAtRoute<Producto>, ValidationProblem, Conflict> Crear(
        CrearProducto dto, ProductoRepository repo)
    {
        var errores = new Dictionary<string, string[]>();
        if (string.IsNullOrWhiteSpace(dto.Nombre)) errores[nameof(dto.Nombre)] = ["El nombre es obligatorio."];
        if (dto.Precio <= 0) errores[nameof(dto.Precio)] = ["El precio debe ser mayor que cero."];
        if (dto.Stock < 0) errores[nameof(dto.Stock)] = ["El stock no puede ser negativo."];
        if (errores.Count > 0) return TypedResults.ValidationProblem(errores);

        if (repo.ExisteNombre(dto.Nombre)) return TypedResults.Conflict();

        var producto = repo.Agregar(dto);
        return TypedResults.CreatedAtRoute(producto, "ObtenerProducto", new { id = producto.Id });
    }

    public static Results<NoContent, NotFound, ProblemHttpResult> Eliminar(int id, ProductoRepository repo)
    {
        var producto = repo.Buscar(id);
        if (producto is null) return TypedResults.NotFound();

        if (producto.Stock > 0)
            return TypedResults.Problem(
                title: "Producto con stock",
                detail: $"Quedan {producto.Stock} unidades; no se puede eliminar.",
                statusCode: StatusCodes.Status409Conflict);

        repo.Eliminar(id);
        return TypedResults.NoContent();
    }
}
```

```text
GET    /api/productos/1   → 200 { "id": 1, "nombre": "Teclado", "precio": 150, "stock": 10 }
GET    /api/productos/9   → 404
POST   /api/productos { "nombre": "", "precio": 0, "stock": -1 }  → 400 con errors.Nombre, errors.Precio, errors.Stock
POST   /api/productos { "nombre": "mouse", "precio": 70, "stock": 3 } → 409 (ya existe)
DELETE /api/productos/1   → 409 ProblemDetails "Producto con stock"
DELETE /api/productos/2   → 204
GET    /openapi/v1.json   → cada endpoint con sus respuestas: 200/404, 201/400/409, 204/404
```

Ninguna de esas respuestas se documentó a mano: OpenAPI las obtuvo de las firmas. La excepción es `ProblemHttpResult`: como su código se decide en tiempo de ejecución, no aporta un código fijo a OpenAPI; si quieres documentar el 409 de `Eliminar`, agrega `.ProducesProblem(StatusCodes.Status409Conflict)` al endpoint.

-----

## Errores comunes

**1. `Unauthorized`, `Forbid` o `Problem` como nombres de tipo.**
Qué pasa: CS0246, el tipo no existe.
Por qué: esos resultados se llaman `UnauthorizedHttpResult`, `ForbidHttpResult` y `ProblemHttpResult`.
Arreglo: mira el tipo que devuelve `TypedResults.X()` (pasa el mouse por encima en el editor) y usa ese.

**2. Mezclar `Results.X()` dentro de `Results<...>`.**
Qué pasa: CS0029: no se puede convertir `IResult` a `Results<Ok<Producto>, NotFound>`.
Por qué: `Results.Ok(x)` devuelve `IResult`, que no es ninguno de los tipos de la unión.
Arreglo: dentro de una unión, siempre `TypedResults`.

**3. Declarar en la unión respuestas que el handler nunca devuelve.**
Qué pasa: el contrato miente, o al revés, OpenAPI muestra un 401 que en realidad produce un filtro o el middleware.
Por qué: se usa la unión para "documentar" algo que hace otro componente.
Arreglo: la unión describe lo que devuelve **el handler**. Lo que agregan los filtros o la autorización se documenta con `.ProducesProblem(401)` o lo infiere `RequireAuthorization()`.

**4. Lambdas largas con Typed Results.**
Qué pasa: firmas ilegibles y handlers imposibles de probar.
Por qué: el tipo de retorno de la lambda se vuelve muy largo.
Arreglo: métodos estáticos con nombre (públicos o `internal` con `InternalsVisibleTo` para las pruebas).

**5. Creer que `TypedResults` mejora el rendimiento.**
Qué pasa: se justifica un refactor por una mejora que no existe.
Por qué: confusión entre "tipado" y "optimizado".
Arreglo: adóptalos por el contrato, OpenAPI y las pruebas.

-----

## Según la versión de .NET

* **.NET 6:** `Results` (estático) e `IResult`.
* **.NET 7:** `TypedResults`, `Results<T1, ..., T6>`, `IEndpointMetadataProvider` (metadatos automáticos para OpenAPI).
* **.NET 8:** `TypedResults` compatibles con el *Request Delegate Generator* y Native AOT.
* **.NET 9:** `Microsoft.AspNetCore.OpenApi` (`AddOpenApi`/`MapOpenApi`) como generador por defecto.
* **.NET 10:** OpenAPI 3.1; `TypedResults.ServerSentEvents` para *server-sent events*.

-----

## Cuándo sí y cuándo no

**Usa Typed Results:**

* Por defecto, en cualquier minimal API nueva: el costo es solo una firma más larga.
* Especialmente si generas clientes desde OpenAPI o pruebas los handlers unitariamente.

**`IResult` sigue siendo útil cuando:**

* Un endpoint puede devolver más de 6 tipos distintos (y probablemente debería dividirse).
* Escribes infraestructura genérica (filtros, helpers) que no conoce los tipos concretos.

-----

## Resumen en 5 líneas

1. `TypedResults.X()` crea el mismo resultado que `Results.X()`, pero con el tipo concreto.
2. `Results<T1, ..., T6>` declara en la firma todas las respuestas posibles; devolver otra no compila.
3. OpenAPI documenta las respuestas automáticamente a partir de esos tipos.
4. Los handlers con nombre que devuelven Typed Results se prueban sin servidor (`resultado.Result`).
5. Los beneficios son contrato, documentación y pruebas, no rendimiento.

-----

## Para profundizar

<details>
<summary>Agregar metadatos que la unión no cubre</summary>

```csharp
grupo.MapGet("/{id:int}", Obtener)
    .WithSummary("Obtiene un producto por id")
    .ProducesProblem(StatusCodes.Status401Unauthorized)   // lo produce la autenticación, no el handler
    .RequireAuthorization();
```

</details>

<details>
<summary>Results con más de 6 tipos</summary>

Si un endpoint necesita más de 6 resultados distintos, casi siempre hace demasiado. Si de verdad hace falta, devuelve `IResult` y documenta con `.Produces<T>(código)`. Ten en cuenta que los errores 4xx/5xx genéricos pueden unificarse con `ProblemHttpResult` (un solo tipo para varios códigos de error).

</details>

-----

## En entrevista

### Respuesta corta (junior)

`TypedResults` es como `Results`, pero devuelve el tipo concreto, por ejemplo `Ok<Producto>` en lugar de `IResult`. Con `Results<Ok<Producto>, NotFound>` en la firma del endpoint declaras qué puede devolver; si intentas devolver otra cosa, no compila. Además, OpenAPI documenta las respuestas solo y es más fácil probar los handlers.

### Respuesta ampliada (semi-senior)

Uso `TypedResults` con uniones `Results<...>` como contrato del handler: el compilador garantiza que el endpoint solo devuelve lo declarado, y como esos tipos implementan `IEndpointMetadataProvider`, OpenAPI se genera sin anotaciones manuales que se desincronicen. Escribo los handlers como métodos estáticos con nombre para probarlos unitariamente verificando `resultado.Result`. No los adopto por rendimiento: crean los mismos objetos que `Results`; las ganancias de las minimal APIs vienen del pipeline por endpoint y de AOT. Lo que agregan los filtros o la autorización lo documento con `ProducesProblem` o lo infiere `RequireAuthorization`, no lo meto en la unión.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `Results.Ok` y `TypedResults.Ok`?**
El tipo estático: `IResult` frente a `Ok<T>`. En tiempo de ejecución es el mismo objeto.

**2. ¿Cuántos tipos admite `Results<...>`?**
De 2 a 6.

**3. ¿Cómo pruebas un handler de minimal API?**
Lo escribo como método con nombre, lo llamo directamente con sus dependencias y verifico el tipo de `resultado.Result`. Para pruebas de integración, `WebApplicationFactory`.

-----

## Práctica

**Ejercicio 1.** ¿Compila? Si no, ¿por qué?

```csharp
static Results<Ok<Producto>, NotFound> Obtener(int id, ProductoRepository repo)
{
    if (id <= 0) return TypedResults.BadRequest();
    return repo.Buscar(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound();
}
```

<details>
<summary>Solución</summary>

No compila: **CS0029**, `BadRequest` no se puede convertir a `Results<Ok<Producto>, NotFound>`. Hay que agregarlo a la unión (`Results<Ok<Producto>, NotFound, BadRequest>`), y así OpenAPI también documenta el 400. Otra opción es la restricción de ruta `{id:int:min(1)}`, que hace que el handler nunca reciba un id inválido (responde 404).

</details>

**Ejercicio 2.** Escribe el handler `ActualizarPrecio(int id, decimal nuevoPrecio, ProductoRepository repo)` con estas respuestas: 404 si no existe, 400 (ValidationProblem) si el precio es ≤ 0, 409 (ProblemDetails) si el nuevo precio supera en más del 50% al actual, y 204 si se actualizó.

<details>
<summary>Solución</summary>

```csharp
public static Results<NoContent, NotFound, ValidationProblem, ProblemHttpResult> ActualizarPrecio(
    int id, decimal nuevoPrecio, ProductoRepository repo)
{
    if (nuevoPrecio <= 0)
        return TypedResults.ValidationProblem(new Dictionary<string, string[]>
        {
            [nameof(nuevoPrecio)] = ["El precio debe ser mayor que cero."]
        });

    var producto = repo.Buscar(id);
    if (producto is null) return TypedResults.NotFound();

    if (nuevoPrecio > producto.Precio * 1.5m)
        return TypedResults.Problem(
            title: "Aumento excesivo",
            detail: $"El precio no puede subir más de un 50% (actual: {producto.Precio}).",
            statusCode: StatusCodes.Status409Conflict);

    repo.Reemplazar(producto with { Precio = nuevoPrecio });   // método a agregar al repositorio
    return TypedResults.NoContent();
}
```

</details>

-----

## Siguiente lección

[Endpoint filters](03-Endpoint%20filters.md)
