# Circuit breaker

## En una frase

Un *circuit breaker* observa fallos y, cuando una dependencia está degradada, rechaza temporalmente nuevas ejecuciones para fallar rápido y darle tiempo de recuperarse.

-----

## Antes de empezar

Conviene que ya sepas:

* Diferenciar fallos transitorios y permanentes: [Reintentos y backoff](01-Reintentos%20y%20backoff.md).
* Manejar código asíncrono y excepciones: [Asincronía](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Circuit breaker:** estrategia con estado que abre el circuito ante demasiados fallos.
* **Fail fast (fallar rápido):** rechazar sin ejecutar la operación remota.
* **Ventana de muestreo:** periodo reciente usado para calcular la tasa de fallos.

-----

## El problema

El servicio de inventario lleva un minuto caído. Cien peticiones por segundo siguen esperando su timeout y consumen conexiones, memoria y tareas:

```text
API de pedidos ──100 llamadas/s──> inventario caído
      │                                  │
      └── cada llamada espera 10 s ──────┘
          hasta 1 000 operaciones pendientes
```

Retry no lo soluciona: puede multiplicar la carga. Tampoco conviene programar `if (fallos == 3)` en cada método. Hace falta una decisión compartida: “esta dependencia está enferma; durante un tiempo no la llamaremos”.

-----

## Cómo funciona

### 1. Tres estados, no tres modos del servidor

```text
                 umbral de fallos
Closed (cerrado) ──────────────────> Open (abierto)
      ▲                                  │
      │ prueba exitosa                   │ termina BreakDuration
      │                                  ▼
      └──────────────────────── Half-Open (medio abierto)
                                  │
                                  └── prueba fallida → Open
```

* **Closed:** deja pasar llamadas y registra sus resultados.
* **Open:** no llama a la dependencia; Polly lanza `BrokenCircuitException`.
* **Half-Open:** permite una ejecución de prueba. El resultado decide si cierra o vuelve a abrir.

El circuito no “comprueba” continuamente el servidor. Cambia de estado a partir de ejecuciones que atraviesan el mismo pipeline.

### 2. Polly 8 usa proporción y volumen mínimo

```csharp
using Polly;
using Polly.CircuitBreaker;

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>(),
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 20,
        BreakDuration = TimeSpan.FromSeconds(15)
    })
    .Build();
```

Se abre si, dentro de 30 segundos, hubo al menos 20 ejecuciones y el 50 % o más falló. `MinimumThroughput` evita abrir por una sola falla en un servicio con poco tráfico.

### 3. El estado debe compartirse en el alcance correcto

Crear el pipeline dentro del método reinicia sus contadores en cada llamada: el circuito nunca aprende. Regístralo una vez por dependencia o cliente lógico. Tampoco compartas el mismo circuito entre servicios independientes: la caída de inventario no debe bloquear pagos.

### 4. Fallar rápido exige una respuesta de negocio

Cuando el circuito está abierto puedes:

* devolver una respuesta degradada o datos cacheados;
* encolar una operación que pueda completarse más tarde;
* responder un error temporal con trazabilidad;
* detener el flujo si continuar sería incorrecto.

El circuit breaker **no repara** la dependencia ni garantiza éxito. Solo limita trabajo condenado a fallar.

-----

## Ejemplo completo

Aplicación de consola con `Polly.Core`. Dos fallos abren el circuito; la tercera ejecución se rechaza sin invocar la operación:

```csharp
using Polly;
using Polly.CircuitBreaker;

var ejecucionesReales = 0;

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>(),
        FailureRatio = 1.0,
        SamplingDuration = TimeSpan.FromSeconds(10),
        MinimumThroughput = 2,
        BreakDuration = TimeSpan.FromSeconds(30),
        OnOpened = _ =>
        {
            Console.WriteLine("Circuito abierto");
            return default;
        }
    })
    .Build();

for (var intento = 1; intento <= 3; intento++)
{
    try
    {
        await pipeline.ExecuteAsync(_ =>
        {
            ejecucionesReales++;
            return ValueTask.FromException(
                new HttpRequestException("Inventario no disponible"));
        });
    }
    catch (BrokenCircuitException)
    {
        Console.WriteLine($"Intento {intento}: rechazado sin llamar");
    }
    catch (HttpRequestException)
    {
        Console.WriteLine($"Intento {intento}: falló la dependencia");
    }
}

Console.WriteLine($"Ejecuciones reales: {ejecucionesReales}");
```

Salida:

```text
Intento 1: falló la dependencia
Circuito abierto
Intento 2: falló la dependencia
Intento 3: rechazado sin llamar
Ejecuciones reales: 2
```

La ejecución que alcanza el umbral todavía recibe el fallo original. Las siguientes reciben `BrokenCircuitException` mientras el circuito siga abierto.

-----

## Errores comunes

**1. Crear un circuito por petición.**
Qué pasa: nunca acumula suficiente historial para abrirse.
Por qué: su estado vive dentro del pipeline.
Arreglo: reutiliza el pipeline o deja que `HttpClientFactory` gestione el handler.

**2. Configurarlo con “tres fallos” copiando Polly 7.**
Qué pasa: se modela mal la API moderna o se abre con muestras poco representativas.
Por qué: Polly 8 usa `FailureRatio`, `SamplingDuration` y `MinimumThroughput`.
Arreglo: calibra proporción, ventana y volumen con métricas reales.

**3. Creer que el circuito reintenta o recupera el servicio.**
Qué pasa: no existe una estrategia para los fallos individuales ni una degradación útil.
Por qué: el breaker solo mide y rechaza.
Arreglo: compón retry cuando corresponda y define qué hará el negocio al fallar rápido.

**4. Un circuito global para todas las dependencias.**
Qué pasa: un proveedor caído bloquea proveedores sanos.
Por qué: comparten estado sin compartir salud.
Arreglo: separa pipelines por destino o partición relevante.

**5. Atrapar `Exception` y devolver datos inventados.**
Qué pasa: se ocultan bugs y el usuario recibe información falsa.
Por qué: se confunde degradación controlada con silenciar errores.
Arreglo: maneja `BrokenCircuitException` explícitamente y usa solo fallbacks válidos.

-----

## Según la versión de .NET

* **Polly 7:** `CircuitBreakerAsync(exceptionsAllowedBeforeBreaking, durationOfBreak)` abría por fallos consecutivos en su forma básica.
* **Polly 8:** `AddCircuitBreaker` modela una tasa de fallos dentro de una ventana y exige un volumen mínimo.
* **.NET 8 a .NET 10:** el handler HTTP estándar ya incluye circuit breaker; se configura mediante `HttpStandardResilienceOptions`.

-----

## Cuándo sí y cuándo no

**Usa circuit breaker cuando:** llamas a una dependencia remota, un fallo sostenido consume recursos y puedes fallar rápido o degradar la función.

**No lo uses cuando:** la operación es local y barata, no hay tráfico suficiente para una muestra útil, o cada petición va a un destino cuya salud es independiente. Para limitar concurrencia usa un *rate limiter*; para limitar duración usa timeout.

-----

## Resumen en 5 líneas

1. El circuit breaker mide fallos y abre temporalmente para fallar rápido.
2. Cerrado ejecuta, abierto rechaza y medio abierto prueba la recuperación.
3. Polly 8 decide con tasa, ventana de muestreo y volumen mínimo.
4. Su pipeline debe reutilizarse para conservar estado.
5. No reintenta ni repara: necesita una respuesta de negocio ante el rechazo.

-----

## Para profundizar

<details>
<summary>Qué observa cuando está dentro de retry</summary>

Las estrategias se envuelven en el orden en que se agregan. Si retry es externo y el breaker interno, el breaker observa cada intento. Si el breaker es externo, observa el resultado final después de agotar retries. No existe un orden universal: decide qué unidad quieres medir y documenta el pipeline.

</details>

<details>
<summary>Aislamiento por destino</summary>

Un cliente que distribuye tráfico entre varios hosts puede necesitar un circuito por autoridad o endpoint. Un único circuito compartido interpretaría fallos de un host como enfermedad de todos. El handler estándar de *hedging* puede mantener grupos de circuitos por destino.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Circuit breaker deja pasar llamadas mientras el servicio parece sano. Si los fallos superan un umbral, abre y rechaza temporalmente. Después permite una prueba y cierra si la dependencia se recuperó.

### Respuesta ampliada (semi-senior)

Es una estrategia con estado y debe compartirse por dependencia lógica. En Polly 8 configuro la tasa de fallos, ventana, throughput mínimo y duración de apertura. Decido si debe observar cada intento o el resultado final según su posición respecto de retry. Instrumento cambios de estado y manejo `BrokenCircuitException` con una degradación válida; no lo presento como un mecanismo de recuperación.

### Preguntas frecuentes de seguimiento

**1. ¿Qué excepción recibe una llamada bloqueada?**
`BrokenCircuitException` o un tipo derivado, no el fallo de la dependencia porque la llamada no ocurrió.

**2. ¿Por qué existe `MinimumThroughput`?**
Para no inferir que el servicio está enfermo a partir de una muestra demasiado pequeña.

**3. ¿Circuit breaker y retry compiten?**
No. Retry trata fallos transitorios individuales; breaker limita daño durante una degradación sostenida.

-----

## Práctica

**Ejercicio 1.** Un servicio recibe dos llamadas por hora. ¿Es razonable `MinimumThroughput = 100` y `SamplingDuration = 30 segundos`?

<details>
<summary>Solución</summary>

No. Nunca alcanzará el volumen mínimo y el circuito no abrirá. La configuración debe corresponder al tráfico real; con tráfico tan bajo quizá baste timeout y manejo explícito del error.

</details>

**Ejercicio 2.** Pagos e inventario usan el mismo pipeline. Pagos falla y también se bloquea inventario. Corrige el diseño.

<details>
<summary>Solución</summary>

Registra un pipeline o cliente tipado por dependencia. Cada uno conserva su propio estado, opciones y telemetría; una caída de pagos no contamina el circuito de inventario.

</details>

-----

## Siguiente lección

[Timeouts y pipelines de resiliencia HTTP](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md)
