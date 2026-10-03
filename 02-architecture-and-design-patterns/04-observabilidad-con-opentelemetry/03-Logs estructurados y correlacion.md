# Logs estructurados y correlación

## En una frase

Un log estructurado conserva un evento como plantilla y atributos consultables; cuando existe una Activity activa, OpenTelemetry puede exportarlo con TraceId y SpanId para unir el detalle del evento con su recorrido.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo se forma una traza: [Trazas distribuidas](01-Trazas%20distribuidas.md).
* Cómo diseñar métricas: [Métricas con Meter](02-Metricas%20con%20Meter.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Log estructurado:** plantilla más atributos tipados.
* **Resource:** identidad del servicio que produce telemetría.
* **Exporter:** componente que entrega señales a otro destino.
* **OTLP:** protocolo neutral para transportar telemetría OpenTelemetry.
* **Observabilidad:** capacidad de explicar el estado del sistema mediante señales.

-----

## El problema

Estos mensajes parecen equivalentes:

```csharp
logger.LogInformation($"Procesando orden {orderId}");
logger.LogInformation("Procesando orden {OrderId}", orderId);
```

No lo son. El primero construye texto y pierde `OrderId` como campo consultable. El segundo conserva una plantilla y un atributo. Aun así, un log con OrderId no responde cuánto tardó cada dependencia ni cuántos pedidos fallan por minuto.

La guía llama “sistema completamente observable” a escribir señales en consola. Eso es DEMASIADO fuerte: consola no aporta almacenamiento durable, consultas, paneles, alertas, retención ni una respuesta operativa.

-----

## Cómo funciona

### 1. ILogger ya es la API de logging de .NET

OpenTelemetry no exige reemplazar `ILogger<T>`:

```csharp
logger.LogInformation("Procesando orden {OrderId} para {Region}", orderId, region);
```

El provider de OpenTelemetry convierte el evento a un `LogRecord`. Conserva categoría, nivel, plantilla y atributos; un exporter decide dónde enviarlo.

### 2. Los scopes agregan contexto común

```csharp
using (logger.BeginScope(new Dictionary<string, object?>
{
    ["tenant.id"] = tenantId,
    ["operation.name"] = "orders.process"
}))
{
    logger.LogInformation("Orden {OrderId} aceptada", orderId);
}
```

Configura `IncludeScopes = true`. No uses scopes para secretos ni para colecciones enormes.

### 3. La correlación con la traza es automática si Activity.Current existe

Un log emitido durante una petición instrumentada recibe TraceId y SpanId del contexto activo. No hace falta agregar manualmente `{TraceId}` a cada plantilla. Puedes mostrarlo durante un diagnóstico, pero el exporter ya tiene campos dedicados y consultables.

Crear una `Activity` aislada solo para obtener un ID es el enfoque equivocado. Instrumenta la operación con `ActivitySource` y deja que el logger lea el contexto.

### 4. El Resource identifica quién emitió la señal

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService(
        serviceName: "Orders.Api",
        serviceVersion: "1.4.0"));
```

Usa la misma identidad de servicio para trazas y métricas. Los logs configurados mediante OpenTelemetry también deben compartir ese recurso para poder consultar todas las señales por servicio y versión.

### 5. Consola enseña; OTLP conecta producción

`AddConsoleExporter()` permite inspeccionar el modelo localmente. En producción, `AddOtlpExporter()` envía a un Collector o backend compatible. El Collector puede procesar, filtrar, enriquecer y enrutar telemetría sin acoplar cada aplicación a un proveedor.

### 6. Correlacionar no significa duplicar todo

| Pregunta | Señal adecuada |
| --- | --- |
| ¿Qué le pasó al pedido 42? | Trace y logs correlacionados |
| ¿Qué porcentaje falla? | Métrica |
| ¿Qué excepción y contexto tuvo este fallo? | Log y span |
| ¿Cumplimos el objetivo de latencia este mes? | Métricas y SLO |

No copies todos los atributos en las tres señales. Conserva solo lo útil, permitido y económicamente sostenible.

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`. Instala `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Exporter.Console`, `OpenTelemetry.Instrumentation.AspNetCore` y `OpenTelemetry.Instrumentation.Runtime`.

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Logging.ClearProviders();
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("Orders.Api"))
    .WithLogging(
        logging => logging.AddConsoleExporter(),
        options =>
        {
            options.IncludeScopes = true;
            options.IncludeFormattedMessage = true;
            options.ParseStateValues = true;
        })
    .WithTracing(tracing => tracing
        .AddSource(Telemetria.SourceName)
        .AddAspNetCoreInstrumentation()
        .AddConsoleExporter())
    .WithMetrics(metrics => metrics
        .AddMeter(Telemetria.MeterName)
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddConsoleExporter());

var app = builder.Build();

app.MapPost("/orders/{id:int}", (int id, ILogger<Program> logger) =>
{
    using var activity = Telemetria.Source.StartActivity("orders.process");
    activity?.SetTag("order.id", id);

    using var scope = logger.BeginScope(new Dictionary<string, object?>
    {
        ["operation.name"] = "orders.process"
    });

    Telemetria.OrdersCreated.Add(1);
    logger.LogInformation("Orden {OrderId} procesada", id);

    return Results.Ok(new { OrderId = id, Status = "processed" });
});

app.Run();

public static class Telemetria
{
    public const string SourceName = "Orders.Api";
    public const string MeterName = "Orders.Api";
    public static readonly ActivitySource Source = new(SourceName);
    private static readonly Meter Meter = new(MeterName);
    public static readonly Counter<long> OrdersCreated =
        Meter.CreateCounter<long>("orders.created", "{order}");
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

El `LogRecord` exportado contiene el cuerpo `Orden 42 procesada`, el atributo `OrderId=42`, el scope `operation.name=orders.process` y los TraceId/SpanId activos. Esos identificadores y timestamps cambian en cada ejecución.

-----

## Errores comunes

**1. Interpolar el mensaje.**
Qué pasa: OrderId queda pegado al texto y es más difícil filtrarlo como campo.
Por qué: la interpolación ocurre antes de llamar al logger.
Arreglo: usa plantilla constante y argumentos separados.

**2. Registrar manualmente TraceId en cada mensaje.**
Qué pasa: duplicación, plantillas ruidosas y posibles inconsistencias.
Por qué: se ignora la correlación del provider con `Activity.Current`.
Arreglo: instrumenta la operación y consulta los campos TraceId/SpanId del log.

**3. Habilitar OpenTelemetry sin exporter o backend.**
Qué pasa: se generan señales, pero nadie las conserva ni consulta.
Por qué: OpenTelemetry no es una base de datos.
Arreglo: configura exportación, Collector/backend, retención y consultas.

**4. Llamar auditoría a cualquier log.**
Qué pasa: se confía en datos que pueden filtrarse, borrarse o modificarse.
Por qué: auditoría exige controles de integridad, acceso, retención y cumplimiento adicionales.
Arreglo: diseña un registro de auditoría específico cuando el negocio o la regulación lo exijan.

**5. Registrar excepciones o cuerpos completos sin redacción.**
Qué pasa: tokens, datos personales o tarjetas llegan al backend.
Por qué: el contexto técnico puede contener datos sensibles.
Arreglo: clasifica, redacta y limita atributos antes de exportar.

**6. Confundir telemetría con observabilidad.**
Qué pasa: existen datos, pero nadie puede responder preguntas ni actuar.
Por qué: faltan SLO, paneles, alertas, responsables y procedimientos.
Arreglo: parte de preguntas operativas y valida que cada señal permita una decisión.

-----

## Según la versión de .NET

* **ASP.NET Core moderno:** `ILogger` crea logs estructurados y `Activity` mantiene el contexto de diagnóstico de la petición.
* **OpenTelemetry .NET:** trazas, métricas y logs están estables, pero sus paquetes se publican con ciclo independiente de .NET.
* **.NET 10:** se conservan `ILogger`, `ActivitySource` y `Meter` como APIs nativas; OpenTelemetry actúa como SDK de recopilación y exportación.

-----

## Cuándo sí y cuándo no

**Usa logs estructurados cuando:** necesitas explicar un evento discreto con contexto consultable, especialmente errores, transiciones importantes y decisiones operativas.

**No registres cuando:** el evento no aporta una pregunta útil, duplica telemetría automática o contiene datos que no puedes proteger. Para tasas y alertas usa métricas; para recorridos y latencia causal usa tracing.

-----

## Resumen en 5 líneas

1. ILogger conserva estructura cuando usas plantillas constantes y argumentos.
2. Los scopes agregan contexto común y deben exportarse explícitamente.
3. Activity.Current permite correlacionar logs con TraceId y SpanId.
4. OpenTelemetry recopila y exporta; no almacena ni vuelve observable al sistema por sí solo.
5. Producción necesita OTLP/backend, seguridad, retención, alertas y responsables.

-----

## Para profundizar

<details>
<summary>OpenTelemetry Collector</summary>

El Collector recibe OTLP, procesa señales y las exporta a uno o varios destinos. Centraliza reintentos, batching, filtrado y enrutamiento. No elimina la necesidad de controlar volumen ni protege automáticamente los datos: sus pipelines también requieren diseño y operación.

</details>

<details>
<summary>Logs y excepciones</summary>

Pasa la excepción como primer argumento para conservar stack trace y tipo:

```csharp
logger.LogError(exception, "No se pudo procesar la orden {OrderId}", orderId);
```

No uses `logger.LogError($"{exception}")`: convierte información estructurada en texto y puede duplicar datos sensibles.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un log estructurado usa una plantilla con propiedades, por ejemplo `OrderId`, en vez de interpolar texto. Si se emite dentro de una Activity, OpenTelemetry agrega TraceId y SpanId para relacionarlo con la traza.

### Respuesta ampliada (semi-senior)

Mantengo `ILogger` como API, uso plantillas constantes y scopes pequeños, y dejo que el provider correlacione con Activity.Current. Configuro un Resource coherente y exporto por OTLP hacia un Collector o backend. Distingo logs operativos de auditoría, aplico redacción y límites de volumen, y no digo que tener tres señales vuelva observable al sistema: hacen falta SLO, consultas, alertas y procesos de respuesta.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no usar interpolación?**
Porque destruye la plantilla y convierte los atributos en texto antes de que el provider los procese.

**2. ¿OpenTelemetry reemplaza ILogger?**
No. Recopila los logs producidos mediante la API estándar de .NET.

**3. ¿ConsoleExporter sirve para producción?**
Es útil para aprendizaje y diagnóstico local. Normalmente se exporta por OTLP a infraestructura dedicada.

-----

## Práctica

**Ejercicio 1.** Corrige este log: `logger.LogInformation($"El usuario {email} creó la orden {orderId}");`.

<details>
<summary>Solución</summary>

```csharp
logger.LogInformation("Orden {OrderId} creada", orderId);
```

La plantilla conserva `OrderId`. Se elimina el email porque no es necesario para la pregunta operativa y contiene información personal.

</details>

**Ejercicio 2.** ¿Tener trazas, métricas y logs en consola vuelve observable al sistema?

<details>
<summary>Solución</summary>

No. Solo demuestra emisión local. Faltan almacenamiento, consultas, correlación verificable, retención, protección de datos, paneles, alertas, SLO y una respuesta humana o automática.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Ejercicios de observabilidad](Ejercicios.md) y vuelve al [índice del módulo](README.md).
