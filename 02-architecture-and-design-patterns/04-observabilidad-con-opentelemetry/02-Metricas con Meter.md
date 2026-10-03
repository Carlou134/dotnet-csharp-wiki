# Métricas con Meter

## En una frase

Una métrica resume muchas observaciones numéricas en series temporales para detectar tendencias, comparar dimensiones y activar alertas sin guardar un evento por cada operación.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué problema resuelven las trazas: [Trazas distribuidas](01-Trazas%20distribuidas.md).
* Clases, campos y ciclos de vida: [POO](../../01-csharp-core-and-runtime/04-poo/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Meter:** fuente con nombre que publica instrumentos.
* **Counter:** total que solo aumenta.
* **Histogram:** distribución de valores observados.
* **Observable gauge:** valor actual leído mediante un callback.
* **Cardinalidad:** cantidad de combinaciones de atributos.

-----

## El problema

Para conocer la latencia promedio, alguien propone buscar millones de logs y calcularla después. Para saber cuántos pedidos están activos, registra una línea al iniciar y otra al terminar y espera que ninguna se pierda.

Eso falla operativamente:

* consultar eventos crudos cuesta más que consultar series agregadas;
* un promedio oculta picos: 99 respuestas rápidas y una de 10 segundos pueden parecer aceptables;
* un identificador por pedido crea una serie distinta por cada valor si se usa como dimensión;
* crear `new Meter("Orders")` y un contador **no exporta nada** si el SDK no registra esa fuente.

-----

## Cómo funciona

### 1. El instrumento responde una clase de pregunta

| Instrumento | Pregunta | Ejemplo |
| --- | --- | --- |
| `Counter<T>` | ¿Cuántas veces ocurrió? | pedidos creados, errores |
| `Histogram<T>` | ¿Cómo se distribuyen los valores? | duración, tamaño |
| `UpDownCounter<T>` | ¿Cuánto subió o bajó acumulativamente? | trabajos activos mediante +1/-1 |
| `ObservableGauge<T>` | ¿Cuál es el valor actual al observar? | profundidad actual de una cola |

Un counter no decrementa. Para duración usa histogram, no un gauge que solo conservaría la última muestra.

### 2. Meter e instrumentos viven tanto como la aplicación

```csharp
private static readonly Meter Meter = new("Orders.Api");
private static readonly Counter<long> Created =
    Meter.CreateCounter<long>("orders.created", unit: "{order}");
```

No crees un meter por petición. Además de desperdiciar recursos, fragmentas la identidad de la instrumentación.

### 3. La fuente debe registrarse

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("Orders.Api")
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddConsoleExporter());
```

`AddMeter` conecta tus instrumentos manuales. Las otras instrumentaciones recopilan métricas de ASP.NET Core y del runtime; no reemplazan tus métricas del negocio.

### 4. Las dimensiones deben tener cardinalidad acotada

```csharp
Created.Add(1, new KeyValuePair<string, object?>("sales.channel", "web"));
```

`sales.channel` puede tener pocos valores conocidos. `order.id`, email, URL completa o texto de excepción pueden generar millones de series y disparar memoria y costo. Esos detalles pertenecen a trazas o logs.

### 5. Diseña nombre, unidad y significado antes del panel

Una métrica es un contrato operativo. `orders.duration` con unidad `ms` no puede cambiar luego a segundos sin romper consultas y alertas. Define qué cuenta, cuándo se incrementa y qué dimensiones admite.

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`. Instala `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.Console` y `OpenTelemetry.Instrumentation.AspNetCore`.

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<OrdersMetrics>();

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("Orders.Api"))
    .WithMetrics(metrics => metrics
        .AddMeter(OrdersMetrics.MeterName)
        .AddAspNetCoreInstrumentation()
        .AddConsoleExporter());

var app = builder.Build();

app.MapPost("/orders", async (OrdersMetrics metrics) =>
{
    var inicio = Stopwatch.GetTimestamp();
    metrics.Started();

    try
    {
        await Task.Delay(20);
        metrics.Created("web");
        return Results.Created("/orders/42", new { Id = 42 });
    }
    finally
    {
        metrics.Finished(Stopwatch.GetElapsedTime(inicio));
    }
});

app.Run();

public sealed class OrdersMetrics : IDisposable
{
    public const string MeterName = "Orders.Api";

    private readonly Meter _meter = new(MeterName);
    private readonly Counter<long> _created;
    private readonly Histogram<double> _duration;
    private int _active;

    public OrdersMetrics()
    {
        _created = _meter.CreateCounter<long>("orders.created", "{order}");
        _duration = _meter.CreateHistogram<double>("orders.duration", "ms");
        _meter.CreateObservableGauge("orders.active", () => Volatile.Read(ref _active), "{order}");
    }

    public void Started() => Interlocked.Increment(ref _active);

    public void Created(string channel) =>
        _created.Add(1, new KeyValuePair<string, object?>("sales.channel", channel));

    public void Finished(TimeSpan duration)
    {
        _duration.Record(duration.TotalMilliseconds);
        Interlocked.Decrement(ref _active);
    }

    public void Dispose() => _meter.Dispose();
}
```

Petición:

```text
POST /orders
```

Respuesta JSON:

```json
{"id":42}
```

En el siguiente ciclo de exportación aparecen `orders.created`, la distribución de `orders.duration` y `orders.active`. El instante de exportación y los límites del histogram dependen del exporter, por eso no se inventa aquí una salida fija.

-----

## Errores comunes

**1. Crear el Meter dentro del endpoint.**
Qué pasa: se crean fuentes e instrumentos repetidamente.
Por qué: se trata la instrumentación como dato por petición.
Arreglo: usa un singleton o campos estáticos de larga vida.

**2. Olvidar AddMeter.**
Qué pasa: el código incrementa, pero OpenTelemetry no recopila la métrica propia.
Por qué: el provider solo escucha las fuentes configuradas.
Arreglo: registra exactamente el nombre usado por `Meter`.

**3. Usar OrderId o UserId como dimensión.**
Qué pasa: explota la cantidad de series y el costo.
Por qué: cada valor distinto multiplica la cardinalidad.
Arreglo: deja identificadores en trazas y logs; usa dimensiones acotadas.

**4. Usar counter para un valor que disminuye.**
Qué pasa: el nombre dice “activos”, pero el total nunca baja.
Por qué: counter es monotónico.
Arreglo: usa `UpDownCounter` para deltas o `ObservableGauge` para observar el valor actual.

**5. Medir solo promedios.**
Qué pasa: se esconden colas largas y usuarios lentos.
Por qué: distribuciones distintas pueden tener el mismo promedio.
Arreglo: registra un histogram y consulta percentiles en el backend.

-----

## Según la versión de .NET

* **.NET 6:** se introduce `System.Diagnostics.Metrics` con `Meter`, counter, histogram e instrumentos observables.
* **Versiones posteriores:** el runtime y ASP.NET Core amplían sus métricas integradas y etiquetas siguiendo convenciones OpenTelemetry.
* **.NET 10:** las APIs `Meter` siguen siendo la base; los paquetes OpenTelemetry se versionan por separado y deben mantenerse compatibles y actualizados.

-----

## Cuándo sí y cuándo no

**Usa métricas cuando:** necesitas tasas, tendencias, percentiles, capacidad o alertas sobre conjuntos de operaciones.

**No uses métricas cuando:** necesitas reconstruir una petición concreta o guardar contexto de alta cardinalidad. Usa una traza o un log, y enlázalos mediante el contexto activo.

-----

## Resumen en 5 líneas

1. Counter cuenta eventos; histogram conserva distribuciones; gauge observa estado actual.
2. Meter e instrumentos deben vivir tanto como la aplicación.
3. AddMeter registra la fuente de métricas personalizadas.
4. Dimensiones de alta cardinalidad multiplican series y costos.
5. Una métrica necesita nombre, unidad y semántica estables.

-----

## Para profundizar

<details>
<summary>Percentiles e histogramas</summary>

El instrumento registra observaciones; el backend agrega buckets y calcula percentiles según su modelo. P50 describe la mediana, mientras P95 o P99 muestran la cola lenta. No calcules percentiles promediando percentiles de instancias: agrega distribuciones compatibles.

</details>

<details>
<summary>Métricas RED y USE</summary>

Para servicios, RED propone tasa (*rate*), errores y duración. Para recursos, USE propone utilización, saturación y errores. Son puntos de partida para responder preguntas; no sustituyen métricas de negocio como pagos aprobados o pedidos abandonados.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las métricas son valores agregados en el tiempo. Uso counter para totales, histogram para duraciones y gauge para estado actual. En .NET se crean con Meter y OpenTelemetry recopila la fuente registrada con AddMeter.

### Respuesta ampliada (semi-senior)

Diseño métricas como contratos estables: nombre, unidad, descripción y dimensiones permitidas. Mantengo Meter e instrumentos durante toda la aplicación, registro la fuente y evito identificadores de alta cardinalidad. Para latencia uso histogram y consulto percentiles; para alertas combino señales técnicas con métricas de negocio. Verifico además costo, temporality y agregación del backend.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no poner OrderId como tag?**
Porque crea una serie por pedido; ese identificador corresponde a una traza o log.

**2. ¿Counter y UpDownCounter son equivalentes?**
No. Counter solo suma valores no negativos; UpDownCounter expresa incrementos y decrementos.

**3. ¿Crear un Meter ya exporta datos?**
No. Hace falta un listener o provider que registre la fuente y un exporter.

-----

## Práctica

**Ejercicio 1.** Elige instrumento para: pagos aprobados, duración de checkout y trabajos actualmente en cola.

<details>
<summary>Solución</summary>

Counter para pagos aprobados, histogram para duración y observable gauge para la profundidad actual de la cola.

</details>

**Ejercicio 2.** Decide cuáles tags aceptarías en `orders.created`: `region`, `sales.channel`, `order.id`, `customer.email`.

<details>
<summary>Solución</summary>

`region` y `sales.channel` si tienen conjuntos pequeños y controlados. `order.id` tiene alta cardinalidad y `customer.email` además contiene datos personales; ambos deben quedar fuera de la métrica.

</details>

-----

## Siguiente lección

[Logs estructurados y correlación](03-Logs%20estructurados%20y%20correlacion.md)
