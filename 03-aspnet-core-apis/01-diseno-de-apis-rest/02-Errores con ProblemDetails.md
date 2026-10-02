# Errores con ProblemDetails

## En una frase

**ProblemDetails** es el formato estándar (RFC 9457) para describir errores en una API HTTP: un JSON con `type`, `title`, `status`, `detail` e `instance` que **todos** los errores de tu API comparten, desde una validación hasta una excepción inesperada.

-----

## Antes de empezar

Conviene que ya sepas:

* Códigos de estado y `[ApiController]`: [Principios REST y códigos de estado](01-Principios%20REST%20y%20codigos%20de%20estado.md).
* Excepciones y excepciones propias: [Lanzar y crear excepciones](../../01-csharp-core-and-runtime/08-excepciones/02-Lanzar%20y%20crear%20excepciones.md).
* Inyección de dependencias por constructor: [Principios DRY KISS y refactoring](../../01-csharp-core-and-runtime/11-codigo-limpio/02-Principios%20DRY%20KISS%20y%20refactoring.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **ProblemDetails:** objeto JSON estándar para describir un error HTTP. Su tipo de contenido es `application/problem+json`.
* **RFC 9457:** el estándar que lo define (2023). Reemplaza al RFC 7807 (2016), con el que es compatible.
* **Middleware:** componente que procesa cada petición y respuesta HTTP en una cadena (*pipeline*).
* **Manejo centralizado de errores:** un único lugar que convierte las excepciones no controladas en respuestas HTTP.
* **Extensiones:** campos adicionales que agregas a un ProblemDetails (`traceId`, `errors`, `codigo`).

-----

## El problema

Tres endpoints de la misma API, tres formatos de error:

```csharp
return BadRequest("Error");                                   // "Error"   (texto plano)
return BadRequest(new { ok = false, msg = "Nombre vacío" });  // { "ok": false, "msg": "..." }
return StatusCode(500, ex.ToString());                        // stack trace completo al cliente
```

Consecuencias:

* El frontend necesita **un parser por endpoint** para mostrar el error.
* `"Error"` no dice **qué** falló ni **cómo** corregirlo.
* El 500 con `ex.ToString()` **filtra** rutas del servidor, nombres de tablas y versiones de librerías: información útil para un atacante.
* Las excepciones que nadie capturó devuelven una página HTML o una respuesta vacía, según el entorno.

-----

## Cómo funciona

### 1. La forma de un ProblemDetails

```json
{
  "type": "https://errores.mitienda.example/pedido-ya-enviado",
  "title": "El pedido ya fue enviado",
  "status": 409,
  "detail": "El pedido 42 se envió el 2026-09-30 y no se puede cancelar.",
  "instance": "POST /api/pedidos/42/cancelar",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

| Campo | Qué es | Regla práctica |
| --- | --- | --- |
| `type` | URI que identifica el **tipo** de problema | Estable; idealmente apunta a documentación. Si se omite, equivale a `about:blank` |
| `title` | Resumen corto del **tipo** de problema | El mismo para todas las ocurrencias del mismo `type` |
| `status` | El código HTTP | Igual al de la respuesta |
| `detail` | Explicación de **esta** ocurrencia | Para humanos; nunca detalles internos |
| `instance` | Identifica **esta** ocurrencia | La ruta de la petición o un Id de incidente |
| *extensiones* | Campos extra | `traceId` para cruzar con los logs, `errors` para validaciones |

El cliente programa contra `status` y `type` (no contra el texto de `title`, que puede traducirse).

### 2. Desde un controlador: `Problem()`

```csharp
return Problem(
    title: "El pedido ya fue enviado",
    detail: $"El pedido {id} no se puede cancelar.",
    statusCode: StatusCodes.Status409Conflict,
    type: "https://errores.mitienda.example/pedido-ya-enviado");
```

⚠️ **Usa argumentos con nombre.** La firma es:

```csharp
Problem(string? detail = null, string? instance = null, int? statusCode = null,
        string? title = null, string? type = null)
```

El **primer** parámetro es `detail`, no `title`. `Problem("ID inválido", statusCode: 400)` pone "ID inválido" en `detail`, y el `title` queda como el genérico "Bad Request".

### 3. Validación: `ValidationProblemDetails`

Con `[ApiController]`, si el modelo no es válido la respuesta 400 se genera **sola**, con un campo `errors` por propiedad:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": { "Cliente": [ "The Cliente field is required." ] },
  "traceId": "00-..."
}
```

Para validaciones que solo puedes hacer en el método:

```csharp
if (request.FechaEntrega < DateOnly.FromDateTime(DateTime.Today))
{
    ModelState.AddModelError(nameof(request.FechaEntrega), "La fecha de entrega no puede estar en el pasado.");
    return ValidationProblem(ModelState);
}
```

### 4. También los errores "vacíos": `NotFound()`, `Forbid()`...

Con `[ApiController]`, los resultados de error sin cuerpo (`NotFound()`, `BadRequest()`, `Conflict()`) se convierten automáticamente en ProblemDetails (la opción `SuppressMapClientErrors` está en `false` por defecto). Y con `app.UseStatusCodePages()`, también los errores que no pasan por un controlador (una ruta inexistente, un 401 del middleware de autenticación).

### 5. Manejo centralizado: `AddProblemDetails` + `IExceptionHandler`

```text
 petición ──► UseExceptionHandler ──► UseStatusCodePages ──► controlador
                     ▲                                             │
                     │      excepción no controlada                │
                     └─────────────────────────────────────────────┘
                     │
                     ▼
           IExceptionHandler(s), en orden de registro
           ¿DomainException?  → 422 + ProblemDetails
           ¿NotFoundException? → 404 + ProblemDetails
           ¿otra?              → 500 + ProblemDetails genérico (detalles solo en el log)
```

```csharp
builder.Services.AddProblemDetails();                              // formato estándar para todo
builder.Services.AddExceptionHandler<ManejadorDeExcepciones>();    // .NET 8+

app.UseExceptionHandler();      // captura las excepciones no controladas
app.UseStatusCodePages();       // cuerpo ProblemDetails para 4xx/5xx sin cuerpo
```

El manejador traduce cada tipo de excepción a un código y un ProblemDetails:

```csharp
sealed class ManejadorDeExcepciones(
    IProblemDetailsService problemDetails,
    ILogger<ManejadorDeExcepciones> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext http, Exception ex, CancellationToken ct)
    {
        var (status, title) = ex switch
        {
            RecursoNoEncontradoException => (StatusCodes.Status404NotFound, "Recurso no encontrado"),
            ReglaDeNegocioException => (StatusCodes.Status422UnprocessableEntity, "Regla de negocio violada"),
            _ => (StatusCodes.Status500InternalServerError, "Error interno del servidor")
        };

        if (status >= 500) logger.LogError(ex, "Error no controlado");
        else logger.LogWarning("{Tipo}: {Mensaje}", ex.GetType().Name, ex.Message);

        http.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = http,
            Exception = ex,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = title,
                Detail = status >= 500 ? "Ocurrió un error inesperado. Usa el traceId para reportarlo." : ex.Message
            }
        });
    }
}
```

Las claves:

* **Las excepciones de dominio** (las del módulo [DDD](../../02-architecture-and-design-patterns/01-domain-driven-design/README.md)) se convierten en 4xx con su mensaje, porque están pensadas para el usuario.
* **Cualquier otra excepción** es un bug: 500 con un mensaje genérico. El detalle real va **al log**, y el `traceId` permite encontrarlo.
* `IProblemDetailsService` escribe con el formato configurado (`application/problem+json`, `traceId`, personalizaciones), igual que el resto de la API.

### 6. Personalizar todos los ProblemDetails a la vez

```csharp
builder.Services.AddProblemDetails(opciones => opciones.CustomizeProblemDetails = ctx =>
{
    ctx.ProblemDetails.Instance ??= $"{ctx.HttpContext.Request.Method} {ctx.HttpContext.Request.Path}";
    ctx.ProblemDetails.Extensions.TryAdd("traceId", Activity.Current?.Id ?? ctx.HttpContext.TraceIdentifier);
});
```

Se aplica a los ProblemDetails del middleware y también a los que generan los controladores (`Problem()`, `ValidationProblem()`, errores automáticos).

### 7. ¿Y el envoltorio `{ success, data }`?

Muchos equipos responden siempre así:

```json
{ "success": true, "data": { ... } }
{ "success": false, "error": "..." }
```

Problemas:

* **`success` repite el código de estado.** Si alguien devuelve `200` con `success: false`, el cliente tiene que mirar los dos.
* **Se mezcla con ProblemDetails:** los errores automáticos (validación, 404, 500) no tienen ese formato, así que el "formato consistente" no lo es.
* Todos los clientes tienen que desenvolver `data` en cada llamada.

La consistencia que importa es: **éxito = código 2xx + el recurso; error = código 4xx/5xx + ProblemDetails**. Un envoltorio solo se justifica para metadatos reales, como la paginación (`{ "items": [...], "page": 2, "totalCount": 135 }`), y aun así se usa solo en las colecciones.

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web`:

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails(opciones => opciones.CustomizeProblemDetails = ctx =>
{
    ctx.ProblemDetails.Instance ??= $"{ctx.HttpContext.Request.Method} {ctx.HttpContext.Request.Path}";
    ctx.ProblemDetails.Extensions.TryAdd("traceId", Activity.Current?.Id ?? ctx.HttpContext.TraceIdentifier);
});
builder.Services.AddExceptionHandler<ManejadorDeExcepciones>();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapControllers();
app.Run();

public class ReglaDeNegocioException(string mensaje) : Exception(mensaje);
public class RecursoNoEncontradoException(string mensaje) : Exception(mensaje);

public record PedidoDto(int Id, string Cliente, string Estado);

[ApiController]
[Route("api/pedidos")]
public class PedidosController : ControllerBase
{
    [HttpGet("{id:int}")]
    public ActionResult<PedidoDto> Obtener(int id)
    {
        if (id <= 0)
            return Problem(
                title: "Id inválido",
                detail: "El id debe ser mayor que cero.",
                statusCode: StatusCodes.Status400BadRequest);

        if (id > 100) return NotFound();                    // ProblemDetails automático
        return new PedidoDto(id, "Ana", id == 42 ? "Enviado" : "Pendiente");
    }

    [HttpPost("{id:int}/cancelar")]
    public IActionResult Cancelar(int id)
    {
        if (id > 100) throw new RecursoNoEncontradoException($"No existe el pedido {id}.");
        if (id == 42) throw new ReglaDeNegocioException("Un pedido enviado no se puede cancelar.");
        return NoContent();
    }

    [HttpGet("falla")]
    public IActionResult Falla() => throw new InvalidOperationException("Cadena de conexión inválida: Server=db01;...");
}

sealed class ManejadorDeExcepciones(
    IProblemDetailsService problemDetails,
    ILogger<ManejadorDeExcepciones> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext http, Exception ex, CancellationToken ct)
    {
        var (status, title) = ex switch
        {
            RecursoNoEncontradoException => (StatusCodes.Status404NotFound, "Recurso no encontrado"),
            ReglaDeNegocioException => (StatusCodes.Status422UnprocessableEntity, "Regla de negocio violada"),
            _ => (StatusCodes.Status500InternalServerError, "Error interno del servidor")
        };

        if (status >= 500) logger.LogError(ex, "Error no controlado");
        else logger.LogWarning("{Tipo}: {Mensaje}", ex.GetType().Name, ex.Message);

        http.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = http,
            Exception = ex,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = title,
                Detail = status >= 500 ? "Ocurrió un error inesperado. Usa el traceId para reportarlo." : ex.Message
            }
        });
    }
}
```

Respuestas (todas con `Content-Type: application/problem+json`; el `traceId` cambia en cada petición):

```text
GET /api/pedidos/0              → 400 { "title": "Id inválido", "status": 400,
                                        "detail": "El id debe ser mayor que cero.",
                                        "instance": "GET /api/pedidos/0", "traceId": "..." }

GET /api/pedidos/500            → 404 { "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
                                        "title": "Not Found", "status": 404, ... }

POST /api/pedidos/42/cancelar   → 422 { "title": "Regla de negocio violada", "status": 422,
                                        "detail": "Un pedido enviado no se puede cancelar.", ... }

GET /api/pedidos/falla          → 500 { "title": "Error interno del servidor", "status": 500,
                                        "detail": "Ocurrió un error inesperado. Usa el traceId para reportarlo.", ... }

GET /api/no-existe              → 404 ProblemDetails (gracias a UseStatusCodePages)
```

Fíjate en el 500: la cadena de conexión de la excepción **no** llega al cliente. Queda en el log, junto al `traceId`.

-----

## Errores comunes

**1. `Problem("mensaje", statusCode: 400)` creyendo que es el título.**
Qué pasa: el mensaje queda en `detail` y el `title` es el genérico "Bad Request".
Por qué: el primer parámetro posicional de `Problem` es `detail`.
Arreglo: argumentos con nombre (`title:`, `detail:`, `statusCode:`).

**2. Escribir el ProblemDetails a mano en `UseExceptionHandler`.**
Qué pasa: falta `Content-Type: application/problem+json`, no hay `traceId` y el formato difiere del resto de la API.
Por qué: `WriteAsJsonAsync` usa `application/json` y no aplica las personalizaciones.
Arreglo: `AddProblemDetails()` + `IExceptionHandler` + `IProblemDetailsService`.

**3. Devolver `ex.Message` o `ex.ToString()` en los 500.**
Qué pasa: fuga de información interna (rutas, cadenas de conexión, consultas SQL).
Por qué: es lo primero que se escribe para "ver el error".
Arreglo: mensaje genérico + `traceId` en la respuesta; la excepción completa al log.

**4. `try/catch` en cada acción del controlador.**
Qué pasa: código repetido y respuestas de error distintas en cada endpoint.
Por qué: no se conoce el manejo centralizado.
Arreglo: deja que la excepción suba y que `IExceptionHandler` la traduzca. Captura en el controlador solo lo que puedes resolver ahí.

**5. Usar excepciones para el flujo normal.**
Qué pasa: código lento y logs llenos de "errores" que no lo son.
Por qué: `throw` por cada validación esperada.
Arreglo: validaciones esperadas → `ValidationProblem` o `Problem` directamente; excepciones de dominio → para reglas violadas en el modelo; `IExceptionHandler` → para lo que se escapa.

**6. `UseExceptionHandler()` sin argumentos y sin servicios registrados.**
Qué pasa: `InvalidOperationException` al iniciar la aplicación.
Por qué: el middleware no sabe cómo responder.
Arreglo: registra `AddProblemDetails()` (o pasa una ruta o un manejador).

-----

## Según la versión de .NET

* **ASP.NET Core 2.1:** clase `ProblemDetails` y errores de validación automáticos con `[ApiController]`.
* **ASP.NET Core 2.2:** los errores de cliente (`NotFound()`, etc.) se convierten en ProblemDetails.
* **.NET 7:** `AddProblemDetails()` e `IProblemDetailsService`; `TypedResults.Problem` en minimal APIs.
* **.NET 8:** `IExceptionHandler`; los enlaces de `type` por defecto apuntan al RFC 9110.
* **.NET 9:** `StatusCodeSelector` en `ExceptionHandlerOptions`, para mapear excepciones a códigos sin escribir un manejador.
* **.NET 10:** cuando un `IExceptionHandler` devuelve `true`, el middleware ya no registra la excepción en los logs por defecto: regístrala tú en el manejador (como en el ejemplo).

-----

## Cuándo sí y cuándo no

**Usa ProblemDetails:**

* En **todas** las respuestas de error de una API HTTP. No hay un buen motivo para inventar otro formato.

**Ten cuidado con:**

* `detail` con información sensible o mensajes técnicos.
* Cambiar los valores de `type` una vez publicados: los clientes programan contra ellos.
* Inventar códigos de estado o usar 500 para errores del cliente.

-----

## Resumen en 5 líneas

1. ProblemDetails (RFC 9457) es el formato estándar de errores: `type`, `title`, `status`, `detail`, `instance` + extensiones.
2. En controladores, `Problem(...)` con argumentos con nombre y `ValidationProblem(ModelState)`.
3. `[ApiController]` ya genera ProblemDetails para validación y para `NotFound()`, `BadRequest()`, etc.
4. Centraliza con `AddProblemDetails()`, `UseExceptionHandler()`, `UseStatusCodePages()` e `IExceptionHandler`.
5. Los 500 nunca exponen detalles internos: mensaje genérico + `traceId`; el resto, al log.

-----

## Para profundizar

<details>
<summary>Un catálogo de tipos de error</summary>

En APIs públicas conviene que cada `type` sea una URL real que documente el problema y cómo resolverlo:

```text
https://errores.mitienda.example/pedido-ya-enviado
https://errores.mitienda.example/stock-insuficiente
https://errores.mitienda.example/pago-rechazado
```

Centralízalos en constantes o en un enum, para no escribir URLs a mano en cada `Problem()`. El cliente hace `switch` sobre el `type`, no sobre el texto.

</details>

<details>
<summary>Result pattern en lugar de excepciones de dominio</summary>

Otra opción es que los casos de uso devuelvan un resultado (`Result<T>` con éxito o error) en lugar de lanzar excepciones, y que el controlador lo traduzca a ProblemDetails. Es más explícito y evita el costo de las excepciones, a cambio de más código en cada capa. Ambos enfoques conviven bien: resultados para errores esperados y `IExceptionHandler` para los inesperados.

</details>

-----

## En entrevista

### Respuesta corta (junior)

ProblemDetails es un formato estándar para los errores de una API: un JSON con `type`, `title`, `status`, `detail` e `instance`. En ASP.NET Core lo genero con `Problem()` en el controlador, y `[ApiController]` lo usa automáticamente en los errores de validación. Así todos los errores tienen la misma forma y el frontend los maneja igual.

### Respuesta ampliada (semi-senior)

Uso ProblemDetails (RFC 9457, sucesor del 7807) para todas las respuestas de error. En controladores, `Problem` y `ValidationProblem`; además, `[ApiController]` mapea los errores de cliente automáticamente. Centralizo con `AddProblemDetails()`, `UseExceptionHandler()` y uno o varios `IExceptionHandler` que traducen excepciones de dominio a 4xx con su mensaje, y cualquier otra a un 500 genérico. Agrego `traceId` e `instance` con `CustomizeProblemDetails` para correlacionar con los logs, y nunca expongo detalles internos. Evito los envoltorios `{ success, data }` porque duplican el código de estado y no son consistentes con los errores automáticos. Para APIs públicas mantengo un catálogo estable de URIs `type`.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `title` y `detail`?**
`title` describe el tipo de problema y es igual en todas sus ocurrencias; `detail` explica esta ocurrencia concreta.

**2. ¿Por qué no devolver el mensaje de la excepción en un 500?**
Porque puede contener información sensible y no le sirve al cliente; se registra en el log y se devuelve un `traceId`.

**3. ¿Cómo manejas los errores globalmente en .NET 8+?**
Con `AddProblemDetails`, `UseExceptionHandler` y clases que implementan `IExceptionHandler`, registradas con `AddExceptionHandler<T>()`.

-----

## Práctica

**Ejercicio 1.** ¿Qué devuelve este código? Corrígelo para que el título sea "Stock insuficiente" y el detalle explique cuántas unidades hay.

```csharp
return Problem("Stock insuficiente", statusCode: 409);
```

<details>
<summary>Solución</summary>

Devuelve un 409 con `title: "Conflict"` y `detail: "Stock insuficiente"`, porque el primer parámetro es `detail`.

```csharp
return Problem(
    title: "Stock insuficiente",
    detail: $"Solo quedan {disponibles} unidades del producto {productoId}.",
    statusCode: StatusCodes.Status409Conflict);
```

</details>

**Ejercicio 2.** Agrega al `ManejadorDeExcepciones` un caso para `UnauthorizedAccessException` → 403 y otro para `OperationCanceledException` cuando el cliente canceló la petición (no hace falta responder: devuelve `true` sin escribir nada y no lo registres como error).

<details>
<summary>Solución</summary>

```csharp
public async ValueTask<bool> TryHandleAsync(HttpContext http, Exception ex, CancellationToken ct)
{
    if (ex is OperationCanceledException && http.RequestAborted.IsCancellationRequested)
        return true;   // el cliente se fue: no hay a quién responder

    var (status, title) = ex switch
    {
        RecursoNoEncontradoException => (StatusCodes.Status404NotFound, "Recurso no encontrado"),
        ReglaDeNegocioException => (StatusCodes.Status422UnprocessableEntity, "Regla de negocio violada"),
        UnauthorizedAccessException => (StatusCodes.Status403Forbidden, "Acceso denegado"),
        _ => (StatusCodes.Status500InternalServerError, "Error interno del servidor")
    };
    // ... igual que antes
}
```

Para el 403 conviene un `detail` genérico ("No tienes permiso para esta operación."), sin `ex.Message`.

</details>

-----

## Siguiente lección

[Versionamiento de APIs](03-Versionamiento%20de%20APIs.md)
