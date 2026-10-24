# Middleware personalizado

## En una frase

Un middleware es una función que recibe cada petición, puede hacer algo **antes** y **después** de llamar a `next` o cortar el recorrido respondiendo él mismo; escribir uno propio exige respetar dos reglas: no tocar la respuesta cuando ya empezó a enviarse y no capturar servicios *scoped* en el constructor.

-----

## Antes de empezar

Conviene que ya sepas:

* El pipeline y su orden de cebolla: [Host, configuración y pipeline HTTP](../00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* Ciclos de vida y dependencias cautivas: [Inyección de dependencias y ciclos de vida](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md).
* Filtros, que se parecen pero actúan en otro lugar: [Endpoint filters](../02-minimal-apis/03-Endpoint%20filters.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Middleware:** componente del pipeline HTTP que procesa la petición y decide si pasarla al siguiente.
* **`RequestDelegate`:** delegado que representa "el resto del pipeline"; llamarlo es `next(context)`.
* **Cortocircuito (*short-circuit*):** responder sin llamar a `next`, de modo que los componentes siguientes no se ejecutan.
* **Respuesta iniciada:** momento en que los encabezados ya se enviaron al cliente (`Response.HasStarted`); desde ahí no se pueden cambiar.

-----

## El problema

Una guía propone este middleware para devolver la hora cuando la URL trae `?time`:

```csharp
public class TimeMiddleware
{
    private readonly RequestDelegate next;
    public TimeMiddleware(RequestDelegate nextRequest) => next = nextRequest;

    public async Task Invoke(HttpContext context)
    {
        await next(context);

        if (context.Request.Query.Any(p => p.Key == "time"))
        {
            await context.Response.WriteAsync(DateTime.Now.ToShortTimeString());
        }
    }
}
```

`GET /api/tareas?time` produce algo así:

```text
[{"id":1,"titulo":"Pagar la luz"}]14:32
```

Lo que pasa:

1. El endpoint **ya respondió** cuando corre el código de después de `next`. La hora se pega al final del JSON: el cliente recibe un cuerpo inválido que no puede deserializar.
2. Si el endpoint había fijado `Content-Length`, escribir de más lanza `InvalidOperationException` y la conexión se corta.
3. Si en lugar de escribir el cuerpo intentara agregar un encabezado, fallaría con `InvalidOperationException: Headers are read-only, response has already started`.
4. `DateTime.Now.ToShortTimeString()` depende de la zona horaria y la cultura del servidor: `14:32` en uno, `2:32 PM` en otro.

Las consecuencias:

* respuestas corruptas para cualquier endpoint que pase por el middleware;
* excepciones que dependen de cómo escribió la respuesta cada endpoint;
* datos que cambian según dónde corre el servidor.

-----

## Cómo funciona

### 1. Antes, después o en lugar de

```text
petición ──► Middleware A: antes ──► Middleware B: antes ──► endpoint
                                                                │
respuesta ◄── Middleware A: después ◄── Middleware B: después ◄─┘
```

| Momento | Qué puedes hacer | Ejemplos |
| --- | --- | --- |
| Antes de `next` | Leer la petición, agregar encabezados de respuesta, decidir si seguir | Correlación, autenticación, límites |
| En lugar de `next` | Responder y cortar el pipeline | Mantenimiento, *health checks*, archivos estáticos |
| Después de `next` | Medir, registrar, leer el código de estado | Tiempos, logs, métricas |

Después de `next`, la respuesta normalmente **ya se está enviando**: puedes leer `StatusCode`, pero no cambiar encabezados ni agregar cuerpo.

### 2. Middleware en línea

```csharp
app.Use(async (context, next) =>
{
    // antes
    await next(context);
    // después
});
```

Útil para cosas chicas o prototipos. Con lógica real, conviene una clase.

### 3. Middleware por convención (la forma más común)

```csharp
public sealed class CorrelacionMiddleware(RequestDelegate next, ILogger<CorrelacionMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context, IUsuarioActual usuario)   // scoped: va como parámetro
    {
        string id = context.Request.Headers["X-Correlation-Id"].FirstOrDefault() ?? Guid.NewGuid().ToString("N");
        context.Response.Headers["X-Correlation-Id"] = id;   // antes de next: la respuesta no empezó

        using (logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = id }))
        {
            await next(context);
        }
    }
}
```

Las reglas de la convención:

* Constructor con `RequestDelegate next` (y servicios **singleton** si los necesitas).
* Método público `Invoke` o `InvokeAsync` que recibe `HttpContext` y devuelve `Task`.
* Los servicios **scoped** se reciben como **parámetros de `InvokeAsync`**, no en el constructor.

¿Por qué? Porque ASP.NET Core crea **una sola instancia** del middleware para toda la aplicación: es un singleton de hecho. Un scoped en su constructor sería una dependencia cautiva. Los parámetros de `InvokeAsync` se resuelven en cada petición, desde su scope.

### 4. `IMiddleware`: una instancia por petición

```csharp
public sealed class AuditoriaMiddleware(AppDbContext db) : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next) { ... }
}

builder.Services.AddScoped<AuditoriaMiddleware>();   // obligatorio: lo crea el contenedor
app.UseMiddleware<AuditoriaMiddleware>();
```

| | Por convención | `IMiddleware` |
| --- | --- | --- |
| Instancias | Una para toda la app | Según el lifetime registrado |
| Scoped en el constructor | ❌ | ✔ |
| Registro en DI | No hace falta | Obligatorio |
| Tipado | Por convención (sin interfaz) | Interfaz explícita |

### 5. Cambiar la respuesta sin romperla

Si necesitas un encabezado con algo que solo sabes al final (como el tiempo total), regístralo para "justo antes de enviar":

```csharp
var inicio = Stopwatch.GetTimestamp();
context.Response.OnStarting(() =>
{
    context.Response.Headers["X-Tiempo-Ms"] = Stopwatch.GetElapsedTime(inicio).TotalMilliseconds.ToString("F0");
    return Task.CompletedTask;
});
await next(context);
```

`OnStarting` se ejecuta en el instante en que la respuesta va a empezar a enviarse, cuando todavía se pueden escribir encabezados.

### 6. Cortocircuito

```csharp
if (context.Request.Query.ContainsKey("hora"))
{
    await context.Response.WriteAsJsonAsync(new { horaUtc = TimeProvider.System.GetUtcNow() });
    return;   // sin next: el resto del pipeline no se ejecuta
}
await next(context);
```

Esta es la versión correcta de la idea del `TimeMiddleware`: responder **en lugar de** pasar al endpoint, con hora UTC en formato ISO 8601 y JSON válido.

### 7. Registrarlo con un método de extensión

```csharp
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseCorrelacion(this IApplicationBuilder app) =>
        app.UseMiddleware<CorrelacionMiddleware>();
}

app.UseCorrelacion();
```

No es "más profesional" por sí mismo: sirve para ocultar detalles de registro (por ejemplo, opciones) y para que la lectura del pipeline sea uniforme con `UseAuthentication()`, `UseCors()`.

### 8. Antes de escribir uno, busca el que ya existe

| Necesidad | Ya existe |
| --- | --- |
| Manejar excepciones | `UseExceptionHandler` + `IExceptionHandler` |
| Registrar peticiones | `AddHttpLogging` / `UseHttpLogging`, o los logs de ASP.NET Core |
| Medir tiempos | Métricas integradas (`http.server.request.duration`) con OpenTelemetry |
| Limitar peticiones | `AddRateLimiter` / `UseRateLimiter` |
| Encabezados de seguridad, CORS, compresión | `UseHsts`, `UseCors`, `UseResponseCompression` |
| Lógica de un endpoint o grupo concreto | Endpoint filters o filtros de MVC |

Middleware es para lo que aplica a **todas** (o casi todas) las peticiones. Si solo aplica a algunos endpoints, un filtro sabe qué endpoint es y qué argumentos recibe.

-----

## Ejemplo completo

Proyecto `dotnet new web`. `Program.cs`:

```csharp
using System.Diagnostics;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseTiempoDeRespuesta();       // 1. el más externo: mide todo lo de adentro

app.Use(async (context, next) =>  // 2. en línea: muestra el orden
{
    Console.WriteLine("→ A");
    await next(context);
    Console.WriteLine("← A");
});

app.Use(async (context, next) =>  // 3. cortocircuito con ?hora
{
    if (context.Request.Query.ContainsKey("hora"))
    {
        Console.WriteLine("  cortocircuito: responde B, el endpoint no se ejecuta");
        await context.Response.WriteAsJsonAsync(new { horaUtc = TimeProvider.System.GetUtcNow() });
        return;
    }

    Console.WriteLine("→ B");
    await next(context);
    Console.WriteLine("← B");
});

app.MapGet("/tareas", () =>
{
    Console.WriteLine("  endpoint /tareas");
    return new[] { new { Id = 1, Titulo = "Pagar la luz" } };
});

app.Run();

public sealed class TiempoDeRespuestaMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        long inicio = Stopwatch.GetTimestamp();

        // El encabezado se escribe justo antes de enviar la respuesta, no después de next
        context.Response.OnStarting(() =>
        {
            context.Response.Headers["X-Tiempo-Ms"] =
                Stopwatch.GetElapsedTime(inicio).TotalMilliseconds.ToString("F0");
            return Task.CompletedTask;
        });

        await next(context);
    }
}

public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseTiempoDeRespuesta(this IApplicationBuilder app) =>
        app.UseMiddleware<TiempoDeRespuestaMiddleware>();
}
```

`GET /tareas` responde:

```text
200  X-Tiempo-Ms: {ms}
     [{"id":1,"titulo":"Pagar la luz"}]
```

Y la consola del servidor muestra:

```text
→ A
→ B
  endpoint /tareas
← B
← A
```

`GET /tareas?hora` responde:

```text
200  X-Tiempo-Ms: {ms}
     {"horaUtc":"{fecha y hora UTC en ISO 8601}"}
```

Y la consola muestra:

```text
→ A
  cortocircuito: responde B, el endpoint no se ejecuta
← A
```

Qué observar:

* El orden de ida (A, B, endpoint) y de vuelta (B, A) es el orden de registro, como capas de cebolla.
* Con `?hora`, B respondió sin llamar a `next`: el endpoint no se ejecutó, pero A sí vio pasar la petición de ida y de vuelta.
* `X-Tiempo-Ms` aparece en las dos respuestas porque se escribe en `OnStarting`, antes de que salgan los encabezados. El valor varía en cada ejecución.
* El JSON del cortocircuito es válido y la hora es UTC: no depende de la zona ni de la cultura del servidor.

-----

## Errores comunes

**1. Escribir en la respuesta después de `next`.**
Qué pasa: cuerpo corrupto (texto pegado al JSON) o `InvalidOperationException` si hay `Content-Length`.
Por qué: el endpoint ya escribió y probablemente envió la respuesta.
Arreglo: responde **en lugar de** llamar a `next` (cortocircuito), o limita el "después" a leer y registrar.

**2. Agregar encabezados cuando la respuesta ya empezó.**
Qué pasa: `InvalidOperationException: Headers are read-only, response has already started`.
Por qué: los encabezados viajan antes que el cuerpo.
Arreglo: escríbelos antes de `next` o en `Response.OnStarting`.

**3. Recibir un servicio scoped en el constructor de un middleware por convención.**
Qué pasa: `InvalidOperationException: Cannot resolve scoped service ... from root provider` al arrancar, o datos compartidos entre peticiones.
Por qué: el middleware se instancia una sola vez para toda la aplicación.
Arreglo: recibe el servicio como parámetro de `InvokeAsync`, o implementa `IMiddleware`.

**4. Olvidar `await next(context)`.**
Qué pasa: las peticiones terminan con `200` y cuerpo vacío, sin llegar a ningún endpoint.
Por qué: no llamar a `next` corta el pipeline, aunque no hayas escrito respuesta.
Arreglo: llama a `next` en todos los caminos que no respondan explícitamente.

**5. Registrar el middleware en el lugar equivocado.**
Qué pasa: un middleware que necesita al usuario autenticado ve `User` vacío; uno que debía atrapar errores no los ve.
Por qué: cada middleware solo ve lo que hicieron los anteriores y solo envuelve a los posteriores.
Arreglo: ubícalo según lo que necesita: después de `UseAuthentication` si lee el usuario; al principio si debe envolver a todos.

-----

## Según la versión de .NET

* **ASP.NET Core 1.0:** middleware por convención con `RequestDelegate` y `app.Use`.
* **ASP.NET Core 2.0:** `IMiddleware` y `IMiddlewareFactory`.
* **ASP.NET Core 6:** `WebApplication`; `app.Use(async (context, next) => ...)` con `next(context)`.
* **ASP.NET Core 7:** `AddRateLimiter` y endpoint filters para Minimal APIs.
* **ASP.NET Core 8:** `IExceptionHandler` y métricas HTTP integradas.
* **.NET 8:** `TimeProvider` para obtener la hora de forma reemplazable en pruebas.

-----

## Cuándo sí y cuándo no

**Escribe un middleware cuando:** la lógica aplica a casi todas las peticiones, no depende del endpoint concreto y no existe ya en el framework (correlación propia, cabeceras corporativas, modo mantenimiento).

**Usa un filtro cuando:** la lógica depende del endpoint, de sus argumentos o de su resultado.

**No escribas un middleware cuando:** ya existe uno integrado (excepciones, CORS, rate limiting, logging HTTP), o la lógica es de negocio y debería estar en un servicio.

-----

## Resumen en 5 líneas

1. Un middleware actúa antes de `next`, después de `next` o en lugar de `next` (cortocircuito).
2. Después de `next` la respuesta ya empezó: lee y registra, pero no escribas cuerpo ni encabezados.
3. Encabezados calculados al final: `Response.OnStarting`.
4. El middleware por convención es una instancia única: los scoped van como parámetros de `InvokeAsync` o con `IMiddleware`.
5. Antes de escribir uno, revisa si el framework ya lo trae.

-----

## Para profundizar

<details>
<summary><code>Map</code>, <code>MapWhen</code> y <code>UseWhen</code>: ramas del pipeline</summary>

`app.Map("/admin", rama => ...)` crea un pipeline aparte para las rutas que empiezan con `/admin`. `app.MapWhen(ctx => condición, rama => ...)` hace lo mismo con cualquier condición; la rama **no vuelve** al pipeline principal. `app.UseWhen(condición, rama => ...)` ejecuta middlewares extra solo cuando se cumple la condición y **sí vuelve** al pipeline principal. Son útiles para aplicar algo a un subconjunto de rutas sin poner `if` dentro del middleware.

</details>

<details>
<summary>Leer el cuerpo de la petición en un middleware</summary>

El cuerpo es un `Stream` que se lee una vez. Si un middleware lo lee (por ejemplo, para auditar), el endpoint lo encuentra vacío. Hay que llamar a `context.Request.EnableBuffering()`, leerlo y volver la posición a 0 (`context.Request.Body.Position = 0`). Además, cuesta memoria: no lo hagas con cuerpos grandes ni en todas las peticiones.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un middleware es una pieza del pipeline que se ejecuta en cada petición, en el orden en que se registra. Puede hacer algo antes y después de llamar a `next` o responder directamente sin llamarlo. Uno propio es una clase con `RequestDelegate` en el constructor y un método `InvokeAsync`.

### Respuesta ampliada (semi-senior)

Distingo tres momentos: antes de `next` (leer la petición, poner encabezados), en lugar de `next` (cortocircuito) y después de `next`, donde la respuesta ya empezó y solo leo o registro; para encabezados calculados al final uso `OnStarting`. El middleware por convención es una única instancia, así que los servicios scoped van como parámetros de `InvokeAsync` o uso `IMiddleware` registrado en DI. Lo ubico en el pipeline según lo que necesita y prefiero los integrados (`IExceptionHandler`, rate limiting, HTTP logging, métricas) antes de escribir uno. Si la lógica depende del endpoint, uso un filtro.

### Preguntas frecuentes de seguimiento

**1. ¿Qué pasa si no llamo a `next`?**
El pipeline se corta: los middlewares siguientes y el endpoint no se ejecutan.

**2. ¿Puedo agregar un encabezado después de `next`?**
Normalmente no, porque la respuesta ya empezó. Usa `OnStarting`.

**3. ¿Cómo uso un `DbContext` en un middleware?**
Como parámetro de `InvokeAsync`, o con un middleware `IMiddleware` registrado como scoped.

**4. ¿Middleware o filtro?**
Middleware para lo que aplica a todas las peticiones; filtro para lo que depende del endpoint.

-----

## Práctica

**Ejercicio 1.** Escribe un middleware de modo mantenimiento: si la opción `Mantenimiento:Activo` es `true`, responde `503` con un ProblemDetails y el encabezado `Retry-After: 120`, salvo para `/salud`.

<details>
<summary>Solución</summary>

```csharp
public sealed class MantenimientoMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context, IOptionsSnapshot<MantenimientoOptions> opciones)
    {
        if (opciones.Value.Activo && !context.Request.Path.StartsWithSegments("/salud"))
        {
            context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
            context.Response.Headers.RetryAfter = "120";
            await context.Response.WriteAsJsonAsync(
                new ProblemDetails { Status = 503, Title = "Servicio en mantenimiento" },
                options: null,
                contentType: "application/problem+json");
            return;
        }
        await next(context);
    }
}

public sealed class MantenimientoOptions
{
    public bool Activo { get; set; }
}

builder.Services.Configure<MantenimientoOptions>(builder.Configuration.GetSection("Mantenimiento"));
app.UseMiddleware<MantenimientoMiddleware>();
```

`IOptionsSnapshot<T>` es scoped (relee la configuración en cada petición), por eso va como parámetro de `InvokeAsync`. Los encabezados se escriben antes de cualquier escritura del cuerpo.

</details>

**Ejercicio 2.** Explica qué imprime la consola del ejemplo si inviertes el orden de A y B (registras primero el de `?hora` y después A) y llamas a `GET /tareas?hora`.

<details>
<summary>Solución</summary>

```text
  cortocircuito: responde B, el endpoint no se ejecuta
```

Solo esa línea. B ahora es el primero en línea después del medidor de tiempo: responde y no llama a `next`, así que A nunca se ejecuta (ni su "→ A" ni su "← A"). El orden de registro decide qué middlewares ven cada petición.

</details>

-----

## Siguiente lección

[Logging con ILogger](04-Logging%20con%20ILogger.md)
