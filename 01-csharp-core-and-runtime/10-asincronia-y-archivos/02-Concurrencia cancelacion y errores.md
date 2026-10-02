# Concurrencia, cancelación y errores

## En una frase

`Task.WhenAll` espera **varias tareas a la vez** (y junta sus resultados), `Task.WhenAny` reacciona a **la primera** que termina, un `CancellationToken` permite **cancelar** operaciones en curso, y las excepciones de una tarea aparecen **al hacer `await`**, así que el `try`/`catch` debe envolver el `await`.

-----

## Antes de empezar

Conviene que ya sepas:

* `async`, `await`, `Task` y `Task<T>`, de [Programación asíncrona](01-Programacion%20asincrona.md).
* `try`/`catch` y filtros `when`, de [Manejo de excepciones](../08-excepciones/01-Manejo%20de%20excepciones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Concurrencia:** varias operaciones en curso al mismo tiempo (aunque no necesariamente ejecutándose en el mismo instante).
* **Paralelismo:** varias operaciones ejecutándose literalmente a la vez, en distintos núcleos del procesador.
* **`Task.WhenAll`:** tarea que termina cuando terminan todas las indicadas.
* **`Task.WhenAny`:** tarea que termina cuando termina la primera de las indicadas.
* **`CancellationTokenSource`:** objeto que **emite** la señal de cancelación.
* **`CancellationToken`:** objeto que **recibe** esa señal; se pasa a los métodos que pueden cancelarse.
* **`OperationCanceledException`:** excepción que indica que una operación se canceló.
* **`AggregateException`:** excepción que agrupa varias excepciones.
* **Tarea con error (*faulted*):** tarea que terminó porque lanzó una excepción.

-----

## El problema

Un panel de control consulta tres servicios: stock, precios y envíos. Con lo que ya sabes, puedes iniciarlos a la vez, pero aparecen preguntas nuevas:

* ¿Cómo espero **los tres** de forma limpia y obtengo los tres resultados?
* El usuario cambia de página: ¿cómo **cancelo** las consultas que ya no sirven?
* El servicio de precios no responde: ¿cómo pongo un **tiempo límite**?
* El servicio de envíos falla: ¿**dónde** se captura esa excepción, si la tarea corría "en segundo plano"?

-----

## Cómo funciona

### Concurrencia no es paralelismo

Iniciar tres llamadas HTTP y esperarlas es **concurrencia**: las tres están "en curso", pero mientras esperan la red no usan el procesador ni ningún hilo. **Paralelismo** es ejecutar cálculos al mismo tiempo en varios núcleos (`Parallel.For`, `Task.Run` con trabajo de CPU). Con `async`/`await` sobre E/S, obtienes concurrencia sin necesidad de hilos extra.

### `Task.WhenAll`: esperar todas

```csharp
Task<int> stock = ConsultarStockAsync();
Task<decimal> precio = ConsultarPrecioAsync();
Task<string> envio = ConsultarEnvioAsync();

await Task.WhenAll(stock, precio, envio);   // espera a que terminen las tres

// ya terminaron: leer .Result aquí NO bloquea (pero await es igual de válido y más claro)
Console.WriteLine($"{await stock} unidades, {await precio:N2}, {await envio}");
```

Si todas las tareas son del mismo tipo, `WhenAll` devuelve directamente un array con los resultados, **en el mismo orden** en que pasaste las tareas:

```csharp
string[] ciudades = { "Lima", "Cusco", "Piura" };

Task<string>[] tareas = ciudades.Select(c => ObtenerClimaAsync(c)).ToArray();
string[] climas = await Task.WhenAll(tareas);   // ["Lima: ...", "Cusco: ...", "Piura: ..."]
```

El tiempo total es el de **la tarea más lenta**, no la suma.

### `Task.WhenAny`: la primera que termine

```csharp
Task<string> espejo1 = DescargarDesdeAsync("servidor-1");
Task<string> espejo2 = DescargarDesdeAsync("servidor-2");

Task<string> ganadora = await Task.WhenAny(espejo1, espejo2);   // devuelve la TAREA que terminó primero
string datos = await ganadora;                                   // su resultado (o su excepción)
```

`WhenAny` devuelve la tarea ganadora, no su resultado: hay que hacerle `await`. Las demás **siguen ejecutándose**: si ya no las necesitas, cancélalas (ver abajo).

Un uso clásico es poner un **tiempo límite**:

```csharp
Task<decimal> consulta = ConsultarPrecioAsync();
Task limite = Task.Delay(TimeSpan.FromSeconds(2));

if (await Task.WhenAny(consulta, limite) == limite)
{
    Console.WriteLine("El servicio de precios tardó demasiado.");
}
else
{
    Console.WriteLine($"Precio: {await consulta:N2}");
}
```

Desde .NET 6 hay una forma más directa: `await consulta.WaitAsync(TimeSpan.FromSeconds(2));`, que lanza `TimeoutException` si se pasa el tiempo.

### Cancelación con `CancellationToken`

La cancelación en .NET es **cooperativa**: quien quiere cancelar emite una señal, y la operación **comprueba** esa señal y se detiene.

```csharp
using var cts = new CancellationTokenSource();          // el emisor
cts.CancelAfter(TimeSpan.FromSeconds(3));                // se cancela solo a los 3 s (opcional)

try
{
    await ProcesarLoteAsync(cts.Token);                  // se pasa el TOKEN, no el source
}
catch (OperationCanceledException)
{
    Console.WriteLine("Proceso cancelado.");
}
```

El método que se puede cancelar recibe el token (por convención, como **último parámetro**, llamado `cancellationToken`) y:

1. Lo **pasa** a todas las operaciones asíncronas que llama (casi todas las APIs de .NET lo aceptan).
2. Lo **comprueba** en los puntos donde tiene sentido detenerse.

```csharp
static async Task ProcesarLoteAsync(CancellationToken cancellationToken = default)
{
    for (int i = 1; i <= 10; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();   // lanza OperationCanceledException si se canceló
        Console.WriteLine($"Procesando elemento {i}");
        await Task.Delay(500, cancellationToken);           // Task.Delay también respeta el token
    }
}
```

Para cancelar manualmente (por ejemplo, cuando el usuario presiona "Cancelar"):

```csharp
cts.Cancel();
```

Si necesitas limpiar antes de salir, consulta `IsCancellationRequested`, limpia y después lanza:

```csharp
if (cancellationToken.IsCancellationRequested)
{
    CerrarArchivosTemporales();
    cancellationToken.ThrowIfCancellationRequested();
}
```

No termines en silencio con un `return` al detectar la cancelación: quien llamó no podría distinguir "terminó bien" de "se canceló". `OperationCanceledException` es la señal estándar.

En ASP.NET Core, cada petición trae un token (`HttpContext.RequestAborted`) que se cancela si el cliente se desconecta; pasarlo a la base de datos evita trabajar para nadie.

### Excepciones en código asíncrono

Las excepciones de un método `async` se **guardan dentro de la `Task`** y se lanzan **al hacer `await`**. Por eso el `try` debe envolver el `await`:

```csharp
Task tarea = FallarAsync();      // la excepción NO aparece aquí: queda guardada en la tarea

try
{
    await tarea;                 // aparece aquí
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Capturada: {ex.Message}");
}

static async Task FallarAsync()
{
    await Task.Delay(100);
    throw new InvalidOperationException("Algo salió mal");
}
```

Una tarea que terminó con una excepción queda en estado **faulted** (`tarea.IsFaulted`), y la excepción se guarda en `tarea.Exception`, que es una `AggregateException` (porque una tarea puede acumular varias).

### Excepciones con `WhenAll`

Si **varias** tareas fallan, `await Task.WhenAll(...)` lanza **solo la primera** excepción. Para verlas todas, inspecciona la tarea de `WhenAll`:

```csharp
Task todas = Task.WhenAll(TareaQueFallaAsync("A"), TareaQueFallaAsync("B"), Task.Delay(100));

try
{
    await todas;
}
catch (Exception primera)
{
    Console.WriteLine($"Primera: {primera.Message}");
    foreach (var ex in todas.Exception!.InnerExceptions)
    {
        Console.WriteLine($"  - {ex.Message}");
    }
}
```

`WhenAll` siempre espera a que **todas** terminen (bien o mal) antes de completarse.

### Limitar la concurrencia

Iniciar 10.000 tareas a la vez contra una API puede saturarla (o hacer que te bloquee). `Parallel.ForEachAsync` (.NET 6) procesa una colección con un máximo de operaciones simultáneas:

```csharp
var opciones = new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = token };

await Parallel.ForEachAsync(urls, opciones, async (url, ct) =>
{
    string contenido = await cliente.GetStringAsync(url, ct);
    Console.WriteLine($"{url}: {contenido.Length} caracteres");
});
```

### Flujos asíncronos: `IAsyncEnumerable<T>`

Cuando los datos llegan de a poco (páginas de una API, filas de una consulta), un iterador asíncrono los entrega a medida que están disponibles:

```csharp
await foreach (var pagina in LeerPaginasAsync(token))
{
    Console.WriteLine($"Página con {pagina.Count} elementos");
}

static async IAsyncEnumerable<List<int>> LeerPaginasAsync(
    [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 3; i++)
    {
        await Task.Delay(300, ct);                  // simula pedir una página a la red
        yield return Enumerable.Range(i * 10, 10).ToList();
    }
}
```

Combina lo visto en [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md) con `await`.

-----

## Ejemplo completo

Un panel que consulta tres servicios a la vez, con tiempo límite y manejo de errores por servicio:

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(1.5));   // límite global

var stock = ConsultarAsync("stock", 400, falla: false, cts.Token);
var precios = ConsultarAsync("precios", 600, falla: true, cts.Token);
var envios = ConsultarAsync("envíos", 3000, falla: false, cts.Token);   // demasiado lento

Task todas = Task.WhenAll(stock, precios, envios);
try
{
    await todas;
}
catch
{
    // los errores se revisan servicio por servicio abajo
}

Mostrar("Stock", stock);
Mostrar("Precios", precios);
Mostrar("Envíos", envios);

static void Mostrar(string nombre, Task<string> tarea)
{
    string estado = tarea.Status switch
    {
        TaskStatus.RanToCompletion => $"OK → {tarea.Result}",
        TaskStatus.Canceled => "cancelado por tiempo límite",
        TaskStatus.Faulted => $"error → {tarea.Exception!.InnerException!.Message}",
        _ => tarea.Status.ToString()
    };
    Console.WriteLine($"{nombre,-8} {estado}");
}

static async Task<string> ConsultarAsync(string servicio, int demoraMs, bool falla, CancellationToken ct)
{
    await Task.Delay(demoraMs, ct);   // si se cancela durante la espera, lanza OperationCanceledException
    if (falla) throw new HttpRequestException($"{servicio} respondió 503");
    return $"datos de {servicio}";
}
```

Salida:

```text
Stock    OK → datos de stock
Precios  error → precios respondió 503
Envíos   cancelado por tiempo límite
```

Cada servicio termina en un estado distinto, y el panel muestra lo que pudo obtener en lugar de fallar por completo. Leer `tarea.Result` aquí es seguro porque se comprobó antes que la tarea terminó correctamente (`RanToCompletion`).

-----

## Errores comunes

**1. Poner el `try` alrededor de la llamada y no del `await`.**
Qué pasa: la excepción no se captura donde esperabas y aparece más tarde (o nunca, si nadie espera la tarea).
Por qué: la excepción se guarda en la `Task` y se lanza al esperarla.
Arreglo: envuelve el `await` con el `try`.

**2. Usar `WhenAny` y olvidarse de las demás tareas.**
Qué pasa: siguen ejecutándose (consumiendo recursos) y sus excepciones nunca se observan.
Por qué: `WhenAny` no cancela nada.
Arreglo: cancela las restantes con un `CancellationToken` cuando ya no las necesitas.

**3. Crear el token y no pasarlo hacia abajo.**
Qué pasa: `cts.Cancel()` no detiene nada.
Por qué: la cancelación es cooperativa: solo se cancela lo que recibe y comprueba el token.
Arreglo: pasa el token a cada método y API asíncrona (`Task.Delay(ms, token)`, `GetAsync(url, token)`...).

**4. Tragarse la cancelación.**
Qué pasa: `catch (Exception)` captura también `OperationCanceledException` y el código trata la cancelación como un error.
Por qué: la cancelación también es una excepción.
Arreglo: captura `OperationCanceledException` por separado, o usa un filtro: `catch (Exception ex) when (ex is not OperationCanceledException)`.

**5. Esperar ver todas las excepciones de `WhenAll` en el `catch`.**
Qué pasa: solo aparece la primera.
Por qué: `await` desenvuelve la `AggregateException` y lanza su primera excepción.
Arreglo: revisa `tareaDeWhenAll.Exception.InnerExceptions`.

**6. No liberar el `CancellationTokenSource`.**
Qué pasa: con `CancelAfter`, quedan temporizadores vivos más de lo necesario.
Por qué: `CancellationTokenSource` es `IDisposable`.
Arreglo: `using var cts = new CancellationTokenSource();`.

-----

## Según la versión de C#

* **.NET 4.0:** `CancellationToken`, `Task.WhenAll`/`WhenAny` (como `Task.Factory.ContinueWhenAll` en sus inicios).
* **.NET 4.5 / C# 5:** `async`/`await` con `WhenAll`/`WhenAny` tal como se usan hoy.
* **C# 8:** `IAsyncEnumerable<T>`, `await foreach` y `await using`.
* **.NET 6:** `Parallel.ForEachAsync` y `Task.WaitAsync(timeout)`.
* **.NET 8:** `CancellationTokenSource.CancelAsync()` y nuevas sobrecargas de `WaitAsync`.
* **.NET 9:** `Task.WhenEach`, que entrega las tareas a medida que van terminando.

-----

## Cuándo sí y cuándo no

**Usa `WhenAll` cuando:**

* Necesitas varios resultados independientes: consultas a distintos servicios, descargas, lecturas.

**Usa `WhenAny` cuando:**

* Te sirve el primero (servidores espejo) o quieres un tiempo límite (aunque `WaitAsync` suele ser más simple).

**Acepta un `CancellationToken` cuando:**

* Tu método asíncrono hace E/S o tarda. Es la convención de .NET y cuesta muy poco agregarlo.

**Limita la concurrencia cuando:**

* Procesas muchas operaciones contra un recurso externo: `Parallel.ForEachAsync` o `SemaphoreSlim`.

-----

## Resumen en 5 líneas

1. Concurrencia (varias operaciones en curso) no es paralelismo (varios cálculos a la vez en distintos núcleos).
2. `await Task.WhenAll(...)` espera todas (y devuelve sus resultados en orden); `Task.WhenAny` devuelve la primera tarea que termina.
3. La cancelación es cooperativa: `CancellationTokenSource` emite, el token se pasa hacia abajo y se comprueba (`ThrowIfCancellationRequested`).
4. Las excepciones de una tarea aparecen al hacer `await`: el `try` envuelve el `await`.
5. Con `WhenAll`, `await` lanza solo la primera excepción; todas están en `.Exception.InnerExceptions`.

-----

## Para profundizar

<details>
<summary>Limitar con SemaphoreSlim</summary>

Antes de `Parallel.ForEachAsync`, el patrón clásico para limitar la concurrencia era un semáforo:

```csharp
var semaforo = new SemaphoreSlim(3);   // como máximo 3 a la vez

var tareas = urls.Select(async url =>
{
    await semaforo.WaitAsync();
    try
    {
        return await cliente.GetStringAsync(url);
    }
    finally
    {
        semaforo.Release();
    }
});

string[] resultados = await Task.WhenAll(tareas);
```

`SemaphoreSlim` también se usa como "lock" asíncrono (con capacidad 1), ya que `lock` no admite `await` dentro.

</details>

<details>
<summary>Canales: productor-consumidor asíncrono</summary>

`System.Threading.Channels` permite que una parte del programa produzca datos y otra los consuma a su ritmo, de forma asíncrona y segura entre hilos:

```csharp
var canal = Channel.CreateBounded<int>(capacity: 10);

var productor = Task.Run(async () =>
{
    for (int i = 0; i < 5; i++) await canal.Writer.WriteAsync(i);
    canal.Writer.Complete();
});

await foreach (int n in canal.Reader.ReadAllAsync())
    Console.WriteLine($"Consumido {n}");
```

Es la base de muchos procesadores en segundo plano (*background workers*).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para ejecutar varias tareas a la vez, se inician todas y se espera con `Task.WhenAll`; `Task.WhenAny` espera a la primera que termine. Para cancelar una operación asíncrona se usa un `CancellationToken`, que se crea con un `CancellationTokenSource` y se pasa a los métodos. Las excepciones de una tarea se lanzan al hacer `await`, por eso el `try/catch` debe rodear el `await`.

### Respuesta ampliada (semi-senior)

`WhenAll` compone tareas y completa cuando todas terminan, propagando en el `await` la primera excepción mientras la `AggregateException` completa queda en `Task.Exception`; `WhenAny` devuelve la tarea ganadora sin cancelar al resto, por lo que se combina con cancelación vinculada (`CreateLinkedTokenSource`). La cancelación es cooperativa: el token se propaga por toda la cadena y se respeta en las APIs de E/S, y se señaliza con `OperationCanceledException`, que conviene distinguir de los errores reales. En servidores, los tokens de petición evitan trabajo inútil. Para cargas grandes se limita la concurrencia con `Parallel.ForEachAsync` o `SemaphoreSlim`, y los flujos se modelan con `IAsyncEnumerable<T>` o canales.

### Preguntas frecuentes de seguimiento

**1. ¿`WhenAll` ejecuta las tareas en paralelo?**
No las ejecuta: espera tareas que ya fueron iniciadas. Si son de E/S, avanzan de forma concurrente sin ocupar hilos mientras esperan.

**2. ¿Cómo se cancela una tarea?**
Con un `CancellationTokenSource`: se pasa su `Token` al método, que debe comprobarlo o pasarlo a las APIs que llama, y se llama a `Cancel()` o `CancelAfter()`.

**3. ¿Qué excepción ves si fallan dos tareas en un `WhenAll`?**
El `await` lanza la primera; todas están en la propiedad `Exception` (una `AggregateException`) de la tarea devuelta por `WhenAll`.

-----

## Práctica

**Ejercicio 1.** Simula 5 descargas con demoras aleatorias entre 200 y 1.000 ms usando `Task.Delay`, lánzalas todas, espera con `WhenAll` y muestra el total de "bytes" descargados (cada descarga devuelve su demora como cantidad de bytes) y el tiempo total.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;

var reloj = Stopwatch.StartNew();
var tareas = Enumerable.Range(1, 5).Select(i => DescargarAsync(i, Random.Shared.Next(200, 1001)));
int[] bytes = await Task.WhenAll(tareas);

Console.WriteLine($"Total: {bytes.Sum()} bytes en {reloj.ElapsedMilliseconds} ms");   // ~ la demora máxima

static async Task<int> DescargarAsync(int id, int demora)
{
    await Task.Delay(demora);
    Console.WriteLine($"Descarga {id} lista ({demora} ms)");
    return demora;
}
```

</details>

**Ejercicio 2.** Escribe un método `ContarAsync(CancellationToken token)` que imprima un número cada 300 ms del 1 al 20. Ejecútalo con un `CancellationTokenSource` que se cancele a los 1.000 ms y muestra "Cancelado en el número X".

<details>
<summary>Solución</summary>

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromMilliseconds(1000));
int ultimo = 0;

try
{
    await ContarAsync(n => ultimo = n, cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine($"Cancelado en el número {ultimo}");   // aproximadamente 3 o 4
}

static async Task ContarAsync(Action<int> alAvanzar, CancellationToken token)
{
    for (int i = 1; i <= 20; i++)
    {
        token.ThrowIfCancellationRequested();
        Console.WriteLine(i);
        alAvanzar(i);
        await Task.Delay(300, token);
    }
}
```

</details>

-----

## Siguiente lección

[Archivos y streams](03-Archivos%20y%20streams.md)
