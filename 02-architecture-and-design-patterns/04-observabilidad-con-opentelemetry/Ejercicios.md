# Ejercicios: observabilidad con OpenTelemetry

Requisitos previos: [Trazas distribuidas](01-Trazas%20distribuidas.md), [Métricas con Meter](02-Metricas%20con%20Meter.md) y [Logs estructurados y correlación](03-Logs%20estructurados%20y%20correlacion.md). Los ejercicios parten de `dotnet new web` y usan los paquetes indicados en cada solución.

-----

## Ejercicios guiados

### Ejercicio 1. Activar tracing en una API

**Objetivo:** recopilar el span servidor de una petición ASP.NET Core.

**Contexto:** la guía mostraba la configuración, pero no indicaba paquetes, identidad del servicio ni que el exportador de consola es una herramienta didáctica y no un backend.

**Instrucciones:** instrumenta ASP.NET Core, identifica el servicio y crea un endpoint determinista.

<details>
<summary>Solución</summary>

Instala `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.Console` y `OpenTelemetry.Instrumentation.AspNetCore`.

```csharp
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("Orders.Api"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddConsoleExporter());

var app = builder.Build();
app.MapGet("/status", () => Results.Ok(new { Status = "ready" }));
app.Run();
```

Petición y respuesta:

```text
GET /status → 200 {"status":"ready"}
```

El exporter muestra un span servidor cuyo nombre de ruta es `GET /status`. TraceId, SpanId, timestamp y duración varían en cada ejecución.

</details>

**Qué observar:** ASP.NET Core ya crea la Activity de la petición; no debes abrir manualmente otro span para representar el mismo trabajo.

### Ejercicio 2. Crear métricas personalizadas

**Objetivo:** contar pedidos y medir su duración con instrumentos apropiados.

**Contexto:** la guía creaba `new Meter("Orders")`, pero no registraba `.AddMeter("Orders")`; por tanto, OpenTelemetry no recopilaría esa métrica propia. También llamaba “en tiempo real” a una señal que normalmente se exporta por intervalos.

**Instrucciones:** crea un singleton con counter e histogram y registra su fuente.

<details>
<summary>Solución</summary>

Instala `OpenTelemetry.Extensions.Hosting` y `OpenTelemetry.Exporter.Console`.

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using OpenTelemetry.Metrics;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<OrderMetrics>();
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m.AddMeter(OrderMetrics.Name).AddConsoleExporter());

var app = builder.Build();
app.MapPost("/orders", async (OrderMetrics metrics) =>
{
    var started = Stopwatch.GetTimestamp();
    await Task.Delay(20);
    metrics.RecordCreated(Stopwatch.GetElapsedTime(started));
    return Results.Ok("created");
});
app.Run();

public sealed class OrderMetrics : IDisposable
{
    public const string Name = "Orders.Api";
    private readonly Meter _meter = new(Name);
    private readonly Counter<long> _created;
    private readonly Histogram<double> _duration;

    public OrderMetrics()
    {
        _created = _meter.CreateCounter<long>("orders.created", "{order}");
        _duration = _meter.CreateHistogram<double>("orders.duration", "ms");
    }

    public void RecordCreated(TimeSpan duration)
    {
        _created.Add(1);
        _duration.Record(duration.TotalMilliseconds);
    }

    public void Dispose() => _meter.Dispose();
}
```

Petición y respuesta:

```text
POST /orders → 200 created
```

En el siguiente intervalo de exportación aparecen `orders.created` y `orders.duration`.

</details>

**Qué observar:** el nombre de `Meter` y el de `AddMeter` deben coincidir exactamente.

### Ejercicio 3. Logging estructurado

**Objetivo:** conservar datos del evento como atributos consultables.

**Contexto:** el ejemplo de la guía era estructurado, pero registrar la hora como propiedad es redundante: todo `LogRecord` ya tiene timestamp. Un evento útil debe describir negocio o diagnóstico, no repetir metadatos automáticos.

**Instrucciones:** registra el identificador del pedido mediante una plantilla constante.

<details>
<summary>Solución</summary>

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapPost("/orders/{id:int}", (int id, ILogger<Program> logger) =>
{
    logger.LogInformation("Orden {OrderId} aceptada", id);
    return Results.Ok("accepted");
});

app.Run();
```

Petición y salida relevante:

```text
POST /orders/42 → 200 accepted
Orden 42 aceptada
```

</details>

**Qué observar:** `OrderId` permanece como propiedad del evento. No uses `$"Orden {id} aceptada"`.

### Ejercicio 4. Correlacionar un log con una traza

**Objetivo:** emitir un log dentro de un span propio y conservar la relación con la petición.

**Contexto:** la guía construía `new Activity("CustomOperation")` y escribía TraceId manualmente. El patrón actual es `ActivitySource`, fuente registrada y correlación automática del provider de logging.

**Instrucciones:** registra una fuente, inicia un span y emite el log sin insertar TraceId en el mensaje.

<details>
<summary>Solución</summary>

Instala `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.Console` y `OpenTelemetry.Instrumentation.AspNetCore`.

```csharp
using System.Diagnostics;
using OpenTelemetry.Logs;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Services.AddOpenTelemetry()
    .WithLogging(logging => logging.AddConsoleExporter())
    .WithTracing(t => t
        .AddSource(Telemetry.SourceName)
        .AddAspNetCoreInstrumentation()
        .AddConsoleExporter());

var app = builder.Build();
app.MapGet("/trace", (ILogger<Program> logger) =>
{
    using var activity = Telemetry.Source.StartActivity("orders.lookup");
    logger.LogInformation("Consultando órdenes pendientes");
    return "Tracing activo";
});
app.Run();

public static class Telemetry
{
    public const string SourceName = "Orders.Api";
    public static readonly ActivitySource Source = new(SourceName);
}
```

Respuesta:

```text
GET /trace → 200 Tracing activo
```

El `LogRecord` contiene los TraceId y SpanId del span `orders.lookup`. Sus valores cambian en cada petición.

</details>

**Qué observar:** la correlación vive en campos dedicados; no hay que contaminar todas las plantillas con identificadores técnicos.

### Ejercicio 5. Instrumentación de las tres señales

**Objetivo:** configurar trazas, métricas y logs con una identidad de servicio coherente.

**Contexto:** la guía llamaba “completamente observable” a exportar tres señales en consola. Esta solución solo demuestra **instrumentación local**; producción todavía requiere OTLP, backend, retención, seguridad, consultas y alertas.

**Instrucciones:** configura un Resource común, tracing y metrics; agrega el provider de logs y expón un endpoint.

<details>
<summary>Solución</summary>

Instala `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.Console`, `OpenTelemetry.Instrumentation.AspNetCore` y `OpenTelemetry.Instrumentation.Runtime`.

```csharp
using System.Diagnostics.Metrics;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("Orders.Api"))
    .WithLogging(
        logging => logging.AddConsoleExporter(),
        options => options.ParseStateValues = true)
    .WithTracing(t => t.AddAspNetCoreInstrumentation().AddConsoleExporter())
    .WithMetrics(m => m
        .AddMeter(Telemetry.MeterName)
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddConsoleExporter());

var app = builder.Build();
app.MapPost("/orders/{id:int}", (int id, ILogger<Program> logger) =>
{
    Telemetry.Created.Add(1);
    logger.LogInformation("Orden {OrderId} creada", id);
    return Results.Created($"/orders/{id}", new { Id = id });
});
app.Run();

public static class Telemetry
{
    public const string MeterName = "Orders.Api";
    private static readonly Meter Meter = new(MeterName);
    public static readonly Counter<long> Created =
        Meter.CreateCounter<long>("orders.created", "{order}");
}
```

Respuesta:

```text
POST /orders/42 → 201 {"id":42}
```

La consola recibe un span HTTP, un log correlacionado y, en su intervalo de exportación, `orders.created`.

</details>

**Qué observar:** emitir las señales es el inicio. La observabilidad aparece cuando esas señales permiten responder preguntas y actuar.

-----

## Retos

### Reto 1. Diagnóstico de latencia

**Misión:** crea un endpoint con spans `orders.validate`, `inventory.reserve` y `payments.authorize`. Usa demoras deterministas distintas e identifica el cuello de botella sin mirar el código.

**Pista:** crea todos los spans con el mismo `ActivitySource`; el span servidor será su ancestro.

<details>
<summary>Solución</summary>

```csharp
app.MapPost("/diagnose", async () =>
{
    await Step("orders.validate", 10);
    await Step("inventory.reserve", 30);
    await Step("payments.authorize", 80);
    return Results.Ok();
});

static async Task Step(string name, int milliseconds)
{
    using var activity = Telemetry.Source.StartActivity(name);
    await Task.Delay(milliseconds);
}
```

Registra `Telemetry.Source` con `AddSource`. La traza mostrará que `payments.authorize` es el tramo más lento. Las duraciones exactas pueden superar levemente las demoras solicitadas por planificación del sistema.

</details>

### Reto 2. Métricas de negocio

**Misión:** mide pedidos aprobados y rechazados, duración del checkout y operaciones activas. Permite segmentar solo por canal de venta.

**Pista:** usa counter, histogram y observable gauge. No uses OrderId como dimensión.

<details>
<summary>Solución</summary>

Crea un singleton similar a `OrdersMetrics` de la lección: counters `orders.approved` y `orders.rejected`, histogram `checkout.duration` en `ms` y gauge `checkout.active`. Cada counter puede recibir `sales.channel` con valores controlados como `web` o `store`. Incrementa activos antes del trabajo y decrementa en `finally` para no dejar un valor falso ante excepciones.

</details>

### Reto 3. Observabilidad completa de un fallo

**Misión:** simula el rechazo de un pago y consigue que una consulta permita encontrar: el span fallido, el log con el error y el incremento de una métrica de rechazos.

**Pista:** comparte Resource y Activity; registra la excepción sin datos sensibles y usa el mismo atributo acotado `payment.reason`.

<details>
<summary>Solución</summary>

Dentro del span `payments.authorize`, marca `ActivityStatusCode.Error`, agrega `payment.reason=insufficient_funds`, incrementa `payments.rejected` con esa dimensión y escribe `logger.LogWarning("Pago rechazado para la orden {OrderId}: {Reason}", orderId, "insufficient_funds")`. El log heredará TraceId y SpanId. En producción exporta por OTLP y valida en el backend que puedas navegar del log a la traza y comparar el evento con la tasa agregada.

</details>

-----

## Checkpoint

1. ¿Por qué `ActivitySource.StartActivity` puede devolver null?
2. ¿Qué ocurre si el nombre de `AddMeter` no coincide con el de `Meter`?
3. ¿Por qué OrderId no debe ser una dimensión de métrica?
4. ¿Qué diferencia hay entre una plantilla de log y una cadena interpolada?
5. ¿Cómo obtiene un log sus TraceId y SpanId?
6. ¿Qué falta después de exportar las tres señales a consola?
