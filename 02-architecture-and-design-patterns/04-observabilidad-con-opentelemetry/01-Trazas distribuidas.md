# Trazas distribuidas

## En una frase

Una traza distribuida reconstruye el recorrido de una operación mediante spans relacionados, para mostrar dónde se consumió tiempo y en qué componente apareció un error.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo entra una petición en una minimal API: [Endpoints y grupos de rutas](../../03-aspnet-core-apis/02-minimal-apis/01-Endpoints%20y%20grupos%20de%20rutas.md).
* Cómo se realizan llamadas con `HttpClient`: [Resiliencia de servicios](../03-resiliencia-de-servicios/03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Trace:** árbol que representa el recorrido completo de una operación.
* **Span:** unidad de trabajo dentro de una traza.
* **Activity / ActivitySource:** API de .NET para instrumentar operaciones trazadas.
* **Sampling (muestreo):** decisión de recopilar solo parte de las trazas.

-----

## El problema

Una petición tarda 2,4 segundos:

```text
cliente → API de pedidos → inventario → pagos → base de datos
                       ¿dónde se gastaron los 2,4 s?
```

Cinco logs aislados con marcas de tiempo no reconstruyen de forma confiable el árbol, los reintentos ni las llamadas concurrentes. Agregar un GUID manual a cada mensaje tampoco conserva automáticamente la relación padre-hijo ni propaga el contexto por HTTP.

Necesitamos una identidad compartida para toda la operación y una identidad distinta para cada tramo medido.

-----

## Cómo funciona

### 1. TraceId relaciona; SpanId distingue

```text
TraceId = a1...9f

HTTP POST /pedidos             SpanId=01  duración=2400 ms
├── validar pedido             SpanId=02  duración=  12 ms
├── GET inventario             SpanId=03  duración= 380 ms
└── POST pagos                 SpanId=04  duración=1950 ms
    └── INSERT transacción     SpanId=05  duración=  25 ms
```

Todos los spans comparten `TraceId`; cada uno tiene su `SpanId` y conoce al padre. La duración crítica aparece sin inferirla a partir de texto.

### 2. ASP.NET Core y HttpClient se instrumentan automáticamente

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddConsoleExporter());
```

`AddAspNetCoreInstrumentation` recopila la actividad creada para la petición entrante. `AddHttpClientInstrumentation` crea spans para llamadas salientes y propaga el contexto W3C (`traceparent`). El proceso remoto debe estar instrumentado para continuar la misma traza.

### 3. El negocio se instrumenta con ActivitySource

```csharp
public static class Telemetria
{
    public const string SourceName = "Orders.Api";
    public static readonly ActivitySource Source = new(SourceName);
}

using var activity = Telemetria.Source.StartActivity("orders.reserve-stock");
activity?.SetTag("product.id", productId);
```

La fuente debe registrarse con `.AddSource(Telemetria.SourceName)`. `StartActivity` puede devolver `null` si no existe listener o la operación no fue muestreada; por eso la instrumentación usa `?.` y nunca controla lógica de negocio.

No uses `new Activity("CustomOperation").Start()` como patrón principal. `ActivitySource` permite suscripción, filtrado, sampling y una identidad estable de la librería que emite el span.

### 4. Atributos y estado explican el span

Usa nombres estables y valores consultables:

```csharp
activity?.SetTag("order.id", orderId);
activity?.SetTag("inventory.reserved", true);
activity?.SetStatus(ActivityStatusCode.Ok);
```

Ante un error esperado, marca el estado y registra la excepción según la convención del SDK. No agregues contraseñas, tokens, cuerpos completos ni datos personales solo porque “ayudan a depurar”.

### 5. El muestreo controla volumen, no la verdad del negocio

Guardar el 100 % de las trazas puede ser caro. Head sampling decide al inicio; tail sampling decide después de observar la traza y puede conservar errores o latencias altas. Una traza descartada no debe cambiar el comportamiento de la aplicación.

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`. Instala `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.Console` y `OpenTelemetry.Instrumentation.AspNetCore`.

```csharp
using System.Diagnostics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("Orders.Api"))
    .WithTracing(tracing => tracing
        .AddSource(Telemetria.SourceName)
        .AddAspNetCoreInstrumentation()
        .AddConsoleExporter());

var app = builder.Build();

app.MapPost("/orders/{id:int}", async (int id) =>
{
    using var activity = Telemetria.Source.StartActivity("orders.process");
    activity?.SetTag("order.id", id);

    await Task.Delay(20);
    activity?.SetStatus(ActivityStatusCode.Ok);

    return Results.Ok(new { OrderId = id, Status = "processed" });
});

app.Run();

public static class Telemetria
{
    public const string SourceName = "Orders.Api";
    public static readonly ActivitySource Source = new(SourceName);
}
```

Petición:

```text
POST /orders/42
```

Respuesta JSON:

```json
{"orderId":42,"status":"processed"}
```

El exportador escribe un span servidor `POST /orders/{id}` y su hijo `orders.process`. Los valores de `TraceId`, `SpanId`, hora y duración cambian en cada ejecución; no se inventan aquí.

-----

## Errores comunes

**1. Crear `new Activity(...)` sin una fuente registrada.**
Qué pasa: la instrumentación manual queda fuera del modelo de suscripción y configuración esperado.
Por qué: se omite `ActivitySource` y `.AddSource(...)`.
Arreglo: define una fuente estable, regístrala y usa `StartActivity`.

**2. Suponer que `StartActivity` nunca devuelve null.**
Qué pasa: aparece `NullReferenceException` cuando no hay listener o el span no fue muestreado.
Por qué: crear telemetría debe ser barato cuando nadie la recopila.
Arreglo: usa `using var activity` y `activity?.SetTag(...)`.

**3. Crear spans para cada método pequeño.**
Qué pasa: ruido, costo y trazas difíciles de leer.
Por qué: se confunde tracing con un profiler.
Arreglo: instrumenta límites remotos y operaciones de negocio relevantes.

**4. Agregar datos sensibles como tags.**
Qué pasa: secretos o información personal llegan a varios sistemas y retenciones.
Por qué: la telemetría suele copiarse y consultarse ampliamente.
Arreglo: usa identificadores permitidos, redacción y una política explícita.

**5. Creer que AddHttpClientInstrumentation instrumenta el servidor remoto.**
Qué pasa: solo aparece el span cliente.
Por qué: la propagación transporta contexto, pero el otro proceso debe recopilar su propia actividad.
Arreglo: instrumenta cada servicio y exporta hacia un backend común.

-----

## Según la versión de .NET

* **.NET 5:** `ActivitySource` y `ActivityListener` establecen el modelo moderno de tracing en .NET.
* **.NET 6 en adelante:** las APIs de diagnóstico del runtime y ASP.NET Core amplían su instrumentación y convenciones.
* **.NET 10:** se mantiene `System.Diagnostics.ActivitySource`; OpenTelemetry recopila esas APIs del framework mediante paquetes versionados independientemente del runtime.

-----

## Cuándo sí y cuándo no

**Usa tracing cuando:** una operación cruza procesos o dependencias, necesitas atribuir latencia, reconstruir causalidad o investigar un fallo concreto.

**No crees spans manuales cuando:** la librería ya está instrumentada o la operación es trivial y de altísima frecuencia. Para tendencias globales usa métricas; para detalles discretos usa logs.

-----

## Resumen en 5 líneas

1. Una traza es un árbol de spans con un TraceId compartido.
2. ASP.NET Core instrumenta entradas y HttpClient instrumenta y propaga salidas.
3. Las operaciones propias se crean con un ActivitySource registrado.
4. StartActivity puede devolver null y la telemetría nunca gobierna el negocio.
5. Sampling, cardinalidad y datos sensibles deben diseñarse antes de producción.

-----

## Para profundizar

<details>
<summary>Propagación W3C</summary>

El encabezado `traceparent` transporta versión, TraceId, SpanId del padre y flags. `tracestate` permite información adicional acordada por proveedores. No construyas estos encabezados a mano: la instrumentación de `HttpClient` inyecta el contexto y ASP.NET Core lo extrae.

</details>

<details>
<summary>Span, evento y log</summary>

Un span mide una operación con inicio y fin. Un evento de span marca algo puntual dentro de esa operación. Un log es una señal independiente que puede correlacionarse con el span activo. No conviertas cada log en un span ni cada paso en un log.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una traza distribuida sigue una petición entre servicios. Todos sus spans comparten TraceId y cada span representa una operación con duración y contexto. En .NET se crean spans propios con ActivitySource y OpenTelemetry los recopila y exporta.

### Respuesta ampliada (semi-senior)

Instrumento automáticamente ASP.NET Core y HttpClient, y uso ActivitySource solo para operaciones de negocio relevantes. Registro la fuente, respeto la propagación W3C y agrego atributos de baja cardinalidad sin datos sensibles. Sé que StartActivity puede devolver null por sampling y mantengo la telemetría fuera de las decisiones del negocio. En producción exporto por OTLP, controlo volumen y verifico que todos los servicios compartan convenciones de recursos.

### Preguntas frecuentes de seguimiento

**1. ¿TraceId y SpanId son lo mismo?**
No. TraceId une toda la operación; SpanId identifica un tramo.

**2. ¿Por qué StartActivity puede devolver null?**
Porque no hay listener interesado o la decisión de sampling descartó la actividad.

**3. ¿OpenTelemetry almacena las trazas?**
No. Las recopila y exporta; un backend las almacena y permite consultarlas.

-----

## Práctica

**Ejercicio 1.** Agrega un span `orders.validate` dentro de un endpoint y etiqueta el resultado sin incluir el email del cliente.

<details>
<summary>Solución</summary>

```csharp
using var validation = Telemetria.Source.StartActivity("orders.validate");
validation?.SetTag("order.valid", true);
validation?.SetStatus(ActivityStatusCode.Ok);
```

El nombre expresa una operación y el tag no filtra información personal.

</details>

**Ejercicio 2.** ¿Debes cancelar un pedido si `StartActivity` devuelve null?

<details>
<summary>Solución</summary>

No. Significa que el span no se recopilará. La instrumentación es una preocupación operativa y no debe decidir la ejecución del negocio.

</details>

-----

## Siguiente lección

[Métricas con Meter](02-Metricas%20con%20Meter.md)
