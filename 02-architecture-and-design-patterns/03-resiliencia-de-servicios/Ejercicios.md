# Ejercicios: resiliencia de servicios con Polly

Requisitos previos: [Reintentos y backoff](01-Reintentos%20y%20backoff.md), [Circuit breaker](02-Circuit%20breaker.md) y [Timeouts y pipelines](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md). Los ejercicios de consola requieren `Polly.Core`; el quinto requiere `Microsoft.Extensions.Http.Resilience`.

-----

## Ejercicios guiados

### Ejercicio 1. Retry con backoff exponencial

**Objetivo:** reintentar un fallo transitorio con un máximo y registrar las repeticiones.

**Contexto:** la guía usaba `api.fail.com`, una dependencia externa no controlada, y solo manejaba `HttpRequestException`; un error HTTP `503` no lanza esa excepción. También omitía jitter.

**Instrucciones:** simula dos fallos, configura tres retries y comprueba cuántas ejecuciones reales hubo.

<details>
<summary>Solución</summary>

```csharp
using Polly;
using Polly.Retry;

var llamadas = 0;
var pipeline = new ResiliencePipelineBuilder<string>()
    .AddRetry(new RetryStrategyOptions<string>
    {
        ShouldHandle = new PredicateBuilder<string>().Handle<HttpRequestException>(),
        MaxRetryAttempts = 3,
        Delay = TimeSpan.Zero, // en producción: demora positiva
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = false,    // en producción: true
        OnRetry = args =>
        {
            Console.WriteLine($"Reintento {args.AttemptNumber + 1}");
            return default;
        }
    })
    .Build();

var resultado = await pipeline.ExecuteAsync(_ =>
{
    llamadas++;
    return llamadas < 3
        ? ValueTask.FromException<string>(new HttpRequestException("Fallo temporal"))
        : ValueTask.FromResult("OK");
});

Console.WriteLine($"Resultado: {resultado}; llamadas: {llamadas}");
```

Salida:

```text
Reintento 1
Reintento 2
Resultado: OK; llamadas: 3
```

</details>

**Qué observar:** `MaxRetryAttempts` no cuenta la ejecución inicial. La demora cero y el jitter desactivado existen solo para una práctica determinista.

### Ejercicio 2. Circuit breaker en acción

**Objetivo:** abrir un circuito y demostrar que una llamada se rechaza sin ejecutar la dependencia.

**Contexto:** la guía usaba la firma de Polly 7 basada en un número de excepciones. Polly 8 decide mediante tasa de fallos, ventana y throughput mínimo. Además, una sola ejecución no alcanza para mostrar el cambio de estado.

**Instrucciones:** abre el circuito después de dos fallos y cuenta las ejecuciones reales.

<details>
<summary>Solución</summary>

```csharp
using Polly;
using Polly.CircuitBreaker;

var ejecuciones = 0;
var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        FailureRatio = 1,
        SamplingDuration = TimeSpan.FromSeconds(10),
        MinimumThroughput = 2,
        BreakDuration = TimeSpan.FromSeconds(30)
    })
    .Build();

for (var n = 1; n <= 3; n++)
{
    try
    {
        await pipeline.ExecuteAsync(_ =>
        {
            ejecuciones++;
            return ValueTask.FromException(new HttpRequestException("Caído"));
        });
    }
    catch (BrokenCircuitException) { Console.WriteLine($"{n}: bloqueado"); }
    catch (HttpRequestException) { Console.WriteLine($"{n}: fallo remoto"); }
}

Console.WriteLine($"Ejecuciones: {ejecuciones}");
```

Salida:

```text
1: fallo remoto
2: fallo remoto
3: bloqueado
Ejecuciones: 2
```

</details>

**Qué observar:** el circuito conserva estado porque se reutiliza. La tercera llamada no llega a la dependencia.

### Ejercicio 3. Timeout cooperativo

**Objetivo:** limitar una operación lenta y detener su trabajo mediante cancelación.

**Contexto:** la guía ejecutaba `Task.Delay(5000)` sin token y afirmaba que la operación se cancelaba automáticamente. Eso confunde abandonar la espera con detener el trabajo.

**Instrucciones:** configura un timeout breve, propaga el token y captura la excepción específica de Polly.

<details>
<summary>Solución</summary>

```csharp
using Polly;
using Polly.Timeout;

var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromMilliseconds(50))
    .Build();

try
{
    await pipeline.ExecuteAsync(async token =>
        await Task.Delay(TimeSpan.FromSeconds(5), token));
}
catch (TimeoutRejectedException)
{
    Console.WriteLine("Tiempo agotado; la espera observó la cancelación.");
}
```

Salida:

```text
Tiempo agotado; la espera observó la cancelación.
```

</details>

**Qué observar:** el token recibido por el callback es el que debe llegar a cada operación cancelable.

### Ejercicio 4. Composición con timeout total y por intento

**Objetivo:** hacer que retry pueda reaccionar al timeout de cada intento sin superar un presupuesto total.

**Contexto:** la guía declaraba un orden sin explicar el anidamiento y usaba `Policy.WrapAsync` de Polly 7. Su timeout interno limitaba cada intento, pero no toda la operación.

**Instrucciones:** crea un pipeline de Polly 8 con timeout total, un retry y timeout por intento.

<details>
<summary>Solución</summary>

```csharp
using Polly;
using Polly.Retry;
using Polly.Timeout;

var intentos = 0;
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(2))
    .AddRetry(new RetryStrategyOptions
    {
        ShouldHandle = new PredicateBuilder().Handle<TimeoutRejectedException>(),
        MaxRetryAttempts = 2,
        Delay = TimeSpan.Zero,
        OnRetry = args =>
        {
            Console.WriteLine($"Reintento {args.AttemptNumber + 1}");
            return default;
        }
    })
    .AddTimeout(TimeSpan.FromMilliseconds(50))
    .Build();

await pipeline.ExecuteAsync(async token =>
{
    intentos++;
    if (intentos < 3)
        await Task.Delay(200, token);
    else
        Console.WriteLine("Operación completada");
});
```

Salida:

```text
Reintento 1
Reintento 2
Operación completada
```

</details>

**Qué observar:** el orden es externo → interno. Retry ve `TimeoutRejectedException` porque el timeout por intento está dentro; el timeout total envuelve todo.

### Ejercicio 5. Integración con HttpClientFactory

**Objetivo:** registrar resiliencia HTTP moderna en el contenedor de dependencias.

**Contexto:** `AddPolicyHandler` pertenece a la integración heredada con Polly 7. En .NET 10 se prefiere `Microsoft.Extensions.Http.Resilience`. Además, reintentar todos los métodos puede duplicar efectos.

**Instrucciones:** registra un cliente tipado con el handler estándar, desactiva retries para métodos inseguros y muestra su dirección base.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;

var services = new ServiceCollection();

services.AddHttpClient<InventarioClient>(client =>
    client.BaseAddress = new Uri("https://inventario.example"))
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.DisableForUnsafeHttpMethods();
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);
    });

await using var provider = services.BuildServiceProvider();
var cliente = provider.GetRequiredService<InventarioClient>();
Console.WriteLine(cliente.BaseAddress);

public sealed class InventarioClient(HttpClient httpClient)
{
    public Uri? BaseAddress => httpClient.BaseAddress;
}
```

Salida:

```text
https://inventario.example/
```

</details>

**Qué observar:** `HttpClientFactory` administra el cliente y el estado de las estrategias. Los valores deben ajustarse con métricas, no copiarse ciegamente.

-----

## Retos

### Reto 1. Cliente HTTP resiliente

**Misión:** crea un cliente tipado para inventario con retry, timeout por intento, logging de repeticiones y cancelación desde el llamador. Usa un handler simulado que falle una vez para que la salida sea reproducible.

**Pista:** parte de `AddResilienceHandler`; agrega retry antes del timeout para que pueda observarlo.

<details>
<summary>Solución</summary>

Combina la estructura del ejemplo completo de [Timeouts y pipelines](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md) con `MaxRetryAttempts = 2`. El cliente debe recibir y propagar un `CancellationToken`; el handler simulado debe devolver `503` una vez y `200` después. La salida esperada es:

```text
Reintento 1 por 503
Stock: 8
```

</details>

### Reto 2. Protección de un servicio crítico

**Misión:** simula diez llamadas, abre el circuito con una tasa de fallos configurable y registra las transiciones abierto, medio abierto y cerrado.

**Pista:** usa `OnOpened`, `OnHalfOpened` y `OnClosed`; emplea una `TimeProvider` controlable en pruebas para no depender de esperas reales.

<details>
<summary>Solución</summary>

Extrae la creación de `CircuitBreakerStrategyOptions` a una función y conserva un solo pipeline. Cuenta por separado llamadas ejecutadas y rechazadas. Primero fuerza suficientes fallos para alcanzar `MinimumThroughput`; avanza el reloj más allá de `BreakDuration`; después devuelve éxito en la ejecución de prueba. Debes observar:

```text
Closed → Open
llamada rechazada
Open → Half-Open
Half-Open → Closed
```

No se fija una duración real en la solución porque dormir el test lo volvería lento y frágil.

</details>

### Reto 3. Pipeline completo y métricas

**Misión:** construye un cliente con timeout total, retry, circuit breaker y timeout por intento. Registra contadores de retries, timeouts, aperturas y rechazos. Comprueba que el máximo de llamadas coincide con el presupuesto definido.

**Pista:** la telemetría debe distinguir una ejecución original de una repetición y una llamada remota de una ejecución rechazada.

<details>
<summary>Solución</summary>

Usa el orden `timeout total → retry → circuit breaker → timeout por intento`. Incrementa contadores desde `OnRetry`, `OnTimeout` y `OnOpened`; cuenta llamadas reales dentro del `HttpMessageHandler` simulado. Con dos retries, ninguna petición puede producir más de tres llamadas remotas. Agrega una aserción:

```csharp
if (llamadasRemotas > 3)
    throw new InvalidOperationException("El pipeline excedió el presupuesto de intentos.");
```

La salida debe informar los contadores; sus valores concretos dependerán del patrón de fallos que definas, por lo que no se inventan aquí.

</details>

-----

## Checkpoint

1. ¿Por qué un `503` no activa una política que solo maneja `HttpRequestException`?
2. ¿Qué diferencia existe entre reintentos e intentos totales?
3. ¿Qué problema resuelve jitter?
4. ¿Por qué el circuit breaker debe reutilizarse?
5. ¿Qué trabajo incluye un timeout total que no incluye uno por intento?
6. ¿Qué debe garantizar el servidor antes de reintentar un `POST` de pago?
