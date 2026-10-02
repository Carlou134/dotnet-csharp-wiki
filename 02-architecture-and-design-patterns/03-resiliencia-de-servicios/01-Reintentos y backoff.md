# Reintentos y backoff

## En una frase

Un *retry* repite una operación cuando el fallo parece transitorio; necesita un límite, una espera y una operación segura de repetir para no convertir una caída pequeña en una sobrecarga o un cobro duplicado.

-----

## Antes de empezar

Conviene que ya sepas:

* Manejar excepciones: [Excepciones](../../01-csharp-core-and-runtime/08-excepciones/README.md).
* Usar `async`, `await` y `CancellationToken`: [Asincronía](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).
* Qué significa idempotencia: [Entrega confiable](../02-event-driven-architecture/03-Entrega%20confiable%20e%20idempotencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Fallo transitorio:** puede desaparecer al esperar y repetir.
* **Fallo permanente:** repetir la misma entrada no lo corrige.
* **Backoff:** espera creciente entre intentos.
* **Jitter:** variación aleatoria que evita reintentos sincronizados.
* **Idempotencia:** repetir no cambia el efecto final.

-----

## El problema

Una API de inventario responde `503 Service Unavailable` durante 200 ms. Abandonar en el primer fallo pierde una venta recuperable. Reintentar sin criterio es peor:

```csharp
while (true)
{
    await httpClient.PostAsync("/pagos", contenido);
}
```

* No hay límite ni espera: el cliente aumenta la carga cuando el servicio menos puede soportarla.
* Un `POST` puede crear varios pagos si el servidor procesó la petición pero la respuesta se perdió.
* Un `400 Bad Request` no se arregla repitiendo los mismos datos.
* `HttpClient.GetAsync` **no lanza** `HttpRequestException` por un `500`; devuelve un `HttpResponseMessage`. Una política que solo maneja la excepción ni siquiera reintentaría ese resultado.

El retry no significa “intentar hasta que funcione”. Significa **clasificar el fallo y gastar un presupuesto acotado**.

-----

## Cómo funciona

### 1. Decide qué errores son transitorios

Para HTTP suelen considerarse candidatos `408`, `429`, algunos `5xx`, `HttpRequestException` y timeouts. No reintentes por defecto:

| Fallo | ¿Reintentar? | Motivo |
| --- | --- | --- |
| Desconexión breve | Sí | La red puede recuperarse |
| `503 Service Unavailable` | Sí, con límite | La dependencia declara indisponibilidad temporal |
| `429 Too Many Requests` | Sí, respetando `Retry-After` | El servidor pide reducir el ritmo |
| `400 Bad Request` | No | La petición seguirá siendo inválida |
| `401 Unauthorized` | No | Requiere corregir credenciales |

### 2. Limita intentos y usa backoff con jitter

Con Polly 8 se construye un `ResiliencePipeline<T>`:

```csharp
using System.Net;
using Polly;
using Polly.Retry;

var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(response => response.StatusCode is
                HttpStatusCode.RequestTimeout or
                HttpStatusCode.TooManyRequests or
                HttpStatusCode.ServiceUnavailable),
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    })
    .Build();
```

`MaxRetryAttempts = 3` significa una ejecución inicial y, como máximo, tres repeticiones: hasta cuatro llamadas. El jitter evita que miles de instancias esperen exactamente 200, 400 y 800 ms y vuelvan juntas.

### 3. Reintenta solo operaciones seguras

`GET`, `HEAD`, `OPTIONS`, `TRACE`, `PUT` y `DELETE` son idempotentes según su semántica HTTP; “idempotente” no significa “sin efectos”. `POST` y `PATCH` no lo son por defecto.

Un pago puede aceptar una clave de idempotencia:

```text
POST /pagos
Idempotency-Key: pedido-845
```

El servidor guarda esa clave junto al resultado. Una repetición devuelve el mismo pago en lugar de crear otro. Sin soporte del servidor, el cliente no puede fabricar idempotencia.

### 4. Registra cada repetición, no cada éxito

`OnRetry` sirve para telemetría. Incluye número de intento, causa, destino y demora; nunca credenciales ni cuerpos sensibles. Un retry que “arregla” todo sin métricas oculta una dependencia degradada.

-----

## Ejemplo completo

Aplicación de consola con el paquete `Polly.Core`. La operación simulada falla dos veces y luego responde:

```csharp
using Polly;
using Polly.Retry;

var llamadas = 0;

var pipeline = new ResiliencePipelineBuilder<string>()
    .AddRetry(new RetryStrategyOptions<string>
    {
        ShouldHandle = new PredicateBuilder<string>()
            .Handle<HttpRequestException>(),
        MaxRetryAttempts = 3,
        Delay = TimeSpan.Zero,
        BackoffType = DelayBackoffType.Constant,
        UseJitter = false,
        OnRetry = args =>
        {
            Console.WriteLine($"Reintento {args.AttemptNumber + 1}: {args.Outcome.Exception!.Message}");
            return default;
        }
    })
    .Build();

var resultado = await pipeline.ExecuteAsync(_ =>
{
    llamadas++;
    return llamadas < 3
        ? ValueTask.FromException<string>(new HttpRequestException("Inventario no disponible"))
        : ValueTask.FromResult("Stock: 8");
});

Console.WriteLine(resultado);
Console.WriteLine($"Llamadas totales: {llamadas}");
```

Salida:

```text
Reintento 1: Inventario no disponible
Reintento 2: Inventario no disponible
Stock: 8
Llamadas totales: 3
```

El ejemplo usa demora cero para que la salida sea inmediata y reproducible. En producción debe existir backoff con jitter.

-----

## Errores comunes

**1. Manejar solo `HttpRequestException` y esperar que un `500` lance una excepción.**
Qué pasa: no hay retry ante respuestas `5xx`.
Por qué: `GetAsync` devuelve normalmente esas respuestas; solo `EnsureSuccessStatusCode()` las convierte en excepción.
Arreglo: maneja el resultado por `StatusCode` o usa las opciones HTTP de `Microsoft.Extensions.Http.Resilience`.

**2. Reintentar cualquier excepción.**
Qué pasa: se repiten errores de programación, validación o autenticación.
Por qué: no se distingue un fallo transitorio de uno permanente.
Arreglo: define predicados estrechos y medidos.

**3. Reintentar un `POST` de pago sin idempotencia.**
Qué pasa: cobros duplicados.
Por qué: el cliente no sabe si el servidor ejecutó la primera petición antes de perderse la respuesta.
Arreglo: clave de idempotencia y almacenamiento atómico en el servidor, o no reintentar.

**4. Backoff exponencial sin jitter.**
Qué pasa: muchas instancias vuelven a llamar al mismo tiempo.
Por qué: todas calculan las mismas esperas.
Arreglo: `UseJitter = true`.

**5. Crear el pipeline dentro de cada llamada.**
Qué pasa: configuración repetida y, para estrategias con estado, estado inútilmente aislado.
Por qué: el pipeline está pensado para reutilizarse.
Arreglo: regístralo una vez con inyección de dependencias o consérvalo durante la vida del cliente.

-----

## Según la versión de .NET

* **Polly 7 y anteriores:** API basada en `Policy.Handle<T>().WaitAndRetryAsync(...)` y `Policy.WrapAsync(...)`.
* **Polly 8:** API basada en `ResiliencePipelineBuilder`, `AddRetry` y opciones fuertemente tipadas; la API 7 se mantiene mediante `Polly` por compatibilidad, pero no es el modelo que se enseña aquí.
* **.NET 8 a .NET 10:** `Microsoft.Extensions.Http.Resilience` integra Polly 8 con `HttpClientFactory` y ofrece handlers estándar y personalizados.

-----

## Cuándo sí y cuándo no

**Usa retry cuando:** el fallo es transitorio, la operación es idempotente o tiene deduplicación, y existe un presupuesto pequeño de intentos y tiempo.

**No lo uses cuando:** el error es permanente, la operación puede duplicar efectos, la dependencia ya está sobrecargada o la latencia adicional viola el presupuesto total. En esos casos convienen fallar, encolar trabajo, degradar funcionalidad o corregir la petición.

-----

## Resumen en 5 líneas

1. Retry sirve para fallos transitorios, no para ocultar cualquier error.
2. Tres reintentos significan hasta cuatro ejecuciones contando la inicial.
3. Backoff reduce la presión y jitter evita que los clientes se sincronicen.
4. Las respuestas HTTP `5xx` no son excepciones por sí solas.
5. Nunca reintentes una operación con efectos sin garantizar idempotencia.

-----

## Para profundizar

<details>
<summary>Retry-After y el retraso calculado por el servidor</summary>

Una respuesta `429` o `503` puede incluir `Retry-After`. Cuando existe y es válido, suele ser mejor respetarlo que inventar una espera local. Polly permite calcular el retraso con `DelayGenerator`; la función debe validar el encabezado y aplicar un máximo para no bloquear el flujo indefinidamente.

</details>

<details>
<summary>El costo multiplicativo</summary>

Si tres capas hacen dos reintentos cada una, una sola operación puede producir hasta `3 × 3 × 3 = 27` llamadas. El retry debe vivir en una capa responsable y su presupuesto debe considerar el timeout total.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Retry repite una operación ante fallos temporales. Debe tener pocos intentos, espera creciente y jitter. Solo se usa si repetir es seguro; de lo contrario puede duplicar efectos.

### Respuesta ampliada (semi-senior)

Primero clasifico excepciones y resultados HTTP. Aplico un presupuesto acotado, backoff exponencial con jitter y telemetría. Respeto `Retry-After` cuando corresponde y no reintento errores de cliente. Para métodos con efectos exijo idempotencia del lado servidor. Además evito retries en varias capas porque multiplican carga y verifico que la suma de intentos y esperas quepa en el timeout total.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué jitter?**
Para evitar que muchas instancias reintenten juntas y provoquen otra sobrecarga.

**2. ¿`HttpRequestException` cubre un `503`?**
No por defecto; `HttpClient` devuelve la respuesta.

**3. ¿Retry mejora siempre la disponibilidad?**
No. Ante una sobrecarga sostenida aumenta tráfico y latencia y puede empeorar la caída.

-----

## Práctica

**Ejercicio 1.** Clasifica estas respuestas: `400`, `408`, `429`, `500`, `503`. Decide cuáles reintentarías y qué información adicional consultarías.

<details>
<summary>Solución</summary>

No se reintenta `400`. `408`, `429`, `500` y `503` pueden ser transitorios, pero deben tener límite. En `429` y `503` se consulta `Retry-After`; para `500` se revisa el contrato de la dependencia, porque no todo error interno es recuperable.

</details>

**Ejercicio 2.** Una API de pagos recibe `POST /pagos`. ¿Alcanza con que el cliente use Polly para repetir de forma segura?

<details>
<summary>Solución</summary>

No. El servidor debe aceptar una clave de idempotencia y guardar atómicamente la clave con el resultado. Si la respuesta se pierde, el retry usa la misma clave y recupera el pago existente.

</details>

-----

## Siguiente lección

[Circuit breaker](02-Circuit%20breaker.md)
