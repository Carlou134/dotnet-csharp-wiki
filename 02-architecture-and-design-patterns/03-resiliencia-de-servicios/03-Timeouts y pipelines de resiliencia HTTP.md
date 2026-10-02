# Timeouts y pipelines de resiliencia HTTP

## En una frase

Un timeout pone un presupuesto de tiempo a una operación; combinado en el orden correcto con retry y circuit breaker forma un pipeline, pero solo detiene de verdad el trabajo que coopera con la cancelación.

-----

## Antes de empezar

Conviene que ya sepas:

* Cancelación cooperativa con `CancellationToken`: [Concurrencia, cancelación y errores](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/02-Concurrencia%20cancelacion%20y%20errores.md).
* Cuándo reintentar: [Reintentos y backoff](01-Reintentos%20y%20backoff.md).
* Cómo protege un circuito: [Circuit breaker](02-Circuit%20breaker.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Timeout por intento:** límite de cada llamada individual.
* **Timeout total:** límite de toda la operación, incluidos retries y esperas.
* **Pipeline de resiliencia:** estrategias ordenadas que envuelven una ejecución.

-----

## El problema

Este límite parece funcionar:

```csharp
var terminada = await Task.WhenAny(OperacionLentaAsync(), Task.Delay(2_000));
```

Pero solo deja de **esperar**. `OperacionLentaAsync` puede seguir usando una conexión, escribiendo datos o cobrando en segundo plano. El mismo error aparece al afirmar que cualquier timeout “cancela automáticamente” la operación.

Además, tres retries con dos segundos por intento pueden tardar ocho segundos, sin contar esperas. Un timeout aislado no define si limita cada intento o el conjunto completo.

-----

## Cómo funciona

### 1. La cancelación es cooperativa

Polly cancela el token entregado al callback. El callback debe propagarlo:

```csharp
await pipeline.ExecuteAsync(
    async token => await httpClient.GetAsync("/stock", token),
    cancellationToken);
```

Si se ignora `token`, Polly puede informar el timeout al llamador, pero el trabajo subyacente podría continuar. No uses timeouts como sustituto de una API cancelable.

### 2. Distingue timeout por intento y total

```text
timeout total 8 s
└── retry
    ├── intento 1 ─ timeout 2 s
    ├── espera
    ├── intento 2 ─ timeout 2 s
    └── intento 3 ─ éxito
```

El timeout interno limita cada intento. El externo incluye intentos y esperas. Sin el externo, el tiempo total puede crecer mucho; sin el interno, un único intento puede gastar todo el presupuesto.

### 3. El orden define quién envuelve a quién

En Polly 8, la primera estrategia agregada es la más externa. Este pipeline permite que retry vea los timeouts de cada intento:

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(8))
    .AddRetry(new RetryStrategyOptions
    {
        ShouldHandle = new PredicateBuilder()
            .Handle<TimeoutRejectedException>(),
        MaxRetryAttempts = 2
    })
    .AddTimeout(TimeSpan.FromSeconds(2))
    .Build();
```

Orden real: timeout total → retry → timeout por intento → operación.

### 4. Para HTTP, parte del handler estándar

Instala `Microsoft.Extensions.Http.Resilience` y registra:

```csharp
builder.Services.AddHttpClient<InventarioClient>(client =>
    client.BaseAddress = new Uri("https://inventario.example"))
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.DisableForUnsafeHttpMethods();
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);
    });
```

El handler estándar compone, de afuera hacia adentro: *rate limiter*, timeout total, retry, circuit breaker y timeout por intento. No apiles varios handlers estándar: si necesitas control fino, usa un único `AddResilienceHandler` personalizado.

-----

## Ejemplo completo

Aplicación de consola. Instala `Microsoft.Extensions.Http.Resilience`. El handler simulado responde `503` dos veces y luego `200`; el pipeline reintenta sin depender de Internet:

```csharp
using System.Net;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http.Resilience;
using Polly;

var services = new ServiceCollection();

services.AddHttpClient<InventarioClient>(client =>
    client.BaseAddress = new Uri("https://inventario.local"))
    .ConfigurePrimaryHttpMessageHandler(() => new InventarioSimuladoHandler())
    .AddResilienceHandler("inventario", pipeline =>
    {
        pipeline.AddTimeout(TimeSpan.FromSeconds(5));
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 2,
            Delay = TimeSpan.Zero,
            BackoffType = DelayBackoffType.Constant,
            UseJitter = false,
            OnRetry = args =>
            {
                Console.WriteLine($"Reintento {args.AttemptNumber + 1} por " +
                    $"{(int)args.Outcome.Result!.StatusCode}");
                return default;
            }
        });
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(15)
        });
        pipeline.AddTimeout(TimeSpan.FromSeconds(1));
    });

await using var provider = services.BuildServiceProvider();
var cliente = provider.GetRequiredService<InventarioClient>();
Console.WriteLine(await cliente.ObtenerStockAsync());

public sealed class InventarioClient(HttpClient httpClient)
{
    public async Task<string> ObtenerStockAsync(CancellationToken cancellationToken = default)
    {
        using var response = await httpClient.GetAsync("/stock/42", cancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(cancellationToken);
    }
}

public sealed class InventarioSimuladoHandler : HttpMessageHandler
{
    private int _llamadas;

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();
        _llamadas++;

        var response = _llamadas < 3
            ? new HttpResponseMessage(HttpStatusCode.ServiceUnavailable)
            : new HttpResponseMessage(HttpStatusCode.OK)
            {
                Content = new StringContent("Stock: 8")
            };

        response.RequestMessage = request;
        return Task.FromResult(response);
    }
}
```

Salida:

```text
Reintento 1 por 503
Reintento 2 por 503
Stock: 8
```

El timeout total es el más externo. Retry envuelve al circuit breaker y al timeout por intento, por lo que cada repetición atraviesa ambos. Las demoras se dejan en cero solo para obtener una salida determinista.

-----

## Errores comunes

**1. No pasar el token a la operación.**
Qué pasa: el llamador recibe un timeout, pero el trabajo puede seguir ejecutándose.
Por qué: la cancelación de .NET es cooperativa.
Arreglo: propaga el token hasta I/O, esperas y dependencias.

**2. Confundir timeout por intento con timeout total.**
Qué pasa: varios retries exceden ampliamente la latencia aceptable.
Por qué: cada intento obtiene un presupuesto nuevo.
Arreglo: usa ambos límites y deja margen entre ellos.

**3. Escribir “Timeout → Retry → Circuit Breaker” sin decir qué es externo.**
Qué pasa: el orden parece claro, pero no explica qué estrategia observa cada resultado.
Por qué: una lista no expresa el anidamiento.
Arreglo: documenta de afuera hacia adentro y dibuja la operación envuelta.

**4. Copiar `Policy.WrapAsync` y `AddPolicyHandler` en código nuevo.**
Qué pasa: se enseña la API de Polly 7 y la integración heredada.
Por qué: la guía no distingue versiones.
Arreglo: en .NET 10 usa `ResiliencePipelineBuilder` y `Microsoft.Extensions.Http.Resilience`.

**5. Apilar handlers de resiliencia.**
Qué pasa: los retries y timeouts se multiplican de manera difícil de razonar.
Por qué: cada handler vuelve a envolver la llamada.
Arreglo: usa un handler estándar o un único handler personalizado.

**6. Reintentar todos los métodos con la configuración estándar sin revisarla.**
Qué pasa: un `POST` puede repetirse.
Por qué: el handler estándar puede reintentar todos los métodos HTTP.
Arreglo: `DisableForUnsafeHttpMethods()` o idempotencia explícita.

-----

## Según la versión de .NET

* **Polly 7:** `TimeoutAsync`, `WaitAndRetryAsync`, `CircuitBreakerAsync` y `Policy.WrapAsync`.
* **Polly 8:** pipelines unificados con `AddTimeout`, `AddRetry` y `AddCircuitBreaker`; el timeout es cooperativo.
* **.NET 8 a .NET 10:** `Microsoft.Extensions.Http.Resilience` ofrece `AddStandardResilienceHandler` y `AddResilienceHandler` sobre Polly 8.

-----

## Cuándo sí y cuándo no

**Usa timeout cuando:** una operación remota tiene un presupuesto de latencia y acepta cancelación. Usa el handler estándar cuando sus valores y estrategias representan bien tu tráfico.

**No agregues otro timeout cuando:** una capa exterior ya impone un presupuesto menor y entiendes que duplicarlo solo confunde el diagnóstico. No uses el pipeline estándar sin ajustar métodos inseguros, latencias y volumen del circuit breaker.

-----

## Resumen en 5 líneas

1. Timeout limita cuánto esperas, pero el trabajo debe cooperar con la cancelación.
2. El timeout por intento y el total resuelven límites diferentes.
3. La primera estrategia agregada por Polly 8 es la más externa.
4. El handler HTTP estándar compone cinco estrategias con valores predeterminados.
5. En .NET 10 se prefieren pipelines de Polly 8 sobre `Policy.WrapAsync` y `AddPolicyHandler`.

-----

## Para profundizar

<details>
<summary>Timeout de HttpClient frente al pipeline</summary>

`HttpClient.Timeout` sigue existiendo, pero combinarlo sin criterio con timeouts de Polly dificulta saber qué límite venció y qué excepción observarás. Configura conscientemente un único presupuesto dominante o establece `HttpClient.Timeout` por encima del timeout total del pipeline.

</details>

<details>
<summary>Valores predeterminados del handler estándar</summary>

El handler estándar incluye límite de concurrencia, timeout total, retry con backoff exponencial y jitter, circuit breaker y timeout por intento. Son un punto de partida, no números universales: una llamada interactiva y un proceso batch no tienen el mismo presupuesto.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Timeout limita la duración de una operación mediante cancelación. En un pipeline suele existir uno por intento y otro total. Se combina con retry y circuit breaker, y el orden determina qué estrategia ve cada fallo.

### Respuesta ampliada (semi-senior)

Propago el token hasta la I/O porque el timeout es cooperativo. Distingo presupuesto total y por intento, y verifico que retries, backoff y latencia quepan dentro del total. Para HTTP en .NET 10 parto de `Microsoft.Extensions.Http.Resilience`, desactivo retries de métodos inseguros salvo que haya idempotencia y uso un solo handler. Instrumento timeouts, retries y cambios del circuito para que la resiliencia no oculte degradaciones.

### Preguntas frecuentes de seguimiento

**1. ¿Qué lanza el timeout de Polly?**
`TimeoutRejectedException` cuando vence el timeout de la estrategia.

**2. ¿El timeout mata un hilo?**
No. Señala cancelación; la operación debe observarla.

**3. ¿Cuál es el orden del handler estándar?**
De afuera hacia adentro: rate limiter, timeout total, retry, circuit breaker y timeout por intento.

-----

## Práctica

**Ejercicio 1.** Tienes timeout por intento de 2 s, dos retries y esperas de 1 s y 2 s. ¿Cuál es el peor tiempo aproximado sin timeout total?

<details>
<summary>Solución</summary>

Hay tres intentos: `3 × 2 s = 6 s`, más `1 s + 2 s` de espera: cerca de 9 s, sin contar sobrecostos. Debe existir un timeout total menor o igual al presupuesto real del usuario.

</details>

**Ejercicio 2.** ¿Por qué `await Task.Delay(5000)` es una mala demostración de cancelación?

<details>
<summary>Solución</summary>

Porque ignora el token. Debe ser `await Task.Delay(5000, cancellationToken)` dentro del callback que recibe el token del pipeline.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Ejercicios de resiliencia](Ejercicios.md) y vuelve al [índice del módulo](README.md).
