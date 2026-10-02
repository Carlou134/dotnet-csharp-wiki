# Ejercicios: asincronía

Cuaderno de práctica extra para el módulo. Los ejercicios guiados repasan cada herramienta con un caso concreto; los retos combinan varias. Intenta resolver cada uno antes de abrir la solución.

Antes de empezar, conviene haber leído [Programación asíncrona](01-Programacion%20asincrona.md) y [Concurrencia, cancelación y errores](02-Concurrencia%20cancelacion%20y%20errores.md).

Todos los programas están escritos con top-level statements: pégalos en un `Program.cs` de un proyecto de consola nuevo.

-----

## Ejercicios guiados

### 1. Bloquear frente a liberar: `Thread.Sleep` y `Task.Delay`

**Objetivo:** ver la diferencia entre congelar un hilo y liberarlo mientras se espera.

**Contexto:** un servicio debe esperar 3 segundos antes de reintentar una conexión fallida.

**Instrucciones:**

1. Espera 3 segundos con `Thread.Sleep(3000)` y mide el tiempo.
2. Espera 3 segundos con `await Task.Delay(3000)` y mide el tiempo.
3. Ahora lanza **5 esperas a la vez** de cada forma y compara cuánto tarda cada versión.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;

var reloj = Stopwatch.StartNew();

// 5 esperas "a la vez" con Thread.Sleep: cada una ocupa un hilo del thread pool mientras duerme
var bloqueantes = Enumerable.Range(1, 5).Select(_ => Task.Run(() => Thread.Sleep(3000)));
await Task.WhenAll(bloqueantes);
Console.WriteLine($"Thread.Sleep x5: {reloj.Elapsed.TotalSeconds:F1} s (5 hilos ocupados sin hacer nada)");

reloj.Restart();

// 5 esperas con Task.Delay: ningún hilo ocupado; solo 5 temporizadores
var asincronas = Enumerable.Range(1, 5).Select(_ => Task.Delay(3000));
await Task.WhenAll(asincronas);
Console.WriteLine($"Task.Delay x5:   {reloj.Elapsed.TotalSeconds:F1} s (0 hilos ocupados durante la espera)");
```

**Qué observar:** los tiempos son parecidos (~3 s), pero la primera versión mantuvo **5 hilos bloqueados**. Con 5.000 esperas en un servidor, `Thread.Sleep` agotaría el thread pool (*thread starvation*), mientras que `Task.Delay` no consume ningún hilo. Todas las APIs de E/S de .NET (`SaveChangesAsync`, `GetStringAsync`, `ReadAllTextAsync`) se comportan como `Task.Delay`: esperan sin ocupar hilos.

</details>

-----

### 2. La promesa de un dato: `Task<T>`

**Objetivo:** obtener un valor de un método asíncrono.

**Contexto:** el backend consulta el perfil de un usuario en la base de datos; la consulta tarda 2 segundos.

**Instrucciones:**

1. Crea `ObtenerPerfilUsuarioAsync(int usuarioId)` que devuelva `Task<string>`.
2. Simula la latencia con `await Task.Delay(2000)`.
3. Devuelve un `string` normal (el compilador lo envuelve en el `Task<string>`).
4. Consúmelo con `await` e imprime el resultado.

<details>
<summary>Solución</summary>

```csharp
Console.WriteLine("Petición recibida. Consultando la base de datos...");
string perfil = await ObtenerPerfilUsuarioAsync(1045);
Console.WriteLine($"Respuesta: {perfil}");

static async Task<string> ObtenerPerfilUsuarioAsync(int usuarioId)
{
    Console.WriteLine($"[SQL] Buscando usuario {usuarioId}...");
    await Task.Delay(2000);                          // latencia simulada
    return $"Usuario_{usuarioId} (Admin)";           // string normal: el compilador arma el Task<string>
}
```

**Qué observar:** `await` "desenvuelve" el `Task<string>` y entrega el `string`. No hace falta tocar `Status` ni `IsCompleted`. Si asignaras la tarea a una variable de tipo `Task` (sin `<string>`), `await` no devolvería nada (error CS0029).

</details>

-----

### 3. Concurrencia: `Task.WhenAll`

**Objetivo:** lanzar varias operaciones de E/S a la vez y esperar solo lo que tarda la más lenta.

**Contexto:** un dashboard financiero carga tres APIs: dólar (2 s), oro (3 s) y bitcoin (1 s). En secuencia serían 6 s.

**Instrucciones:**

1. Invoca los tres métodos **sin** `await` y guarda cada `Task<string>`.
2. Espéralos juntos con `await Task.WhenAll(...)`.
3. Muestra los tres valores y el tiempo total.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;

var reloj = Stopwatch.StartNew();

Task<string> dolar = ObtenerDolarAsync();      // arrancan las tres
Task<string> oro = ObtenerOroAsync();
Task<string> bitcoin = ObtenerBitcoinAsync();

await Task.WhenAll(dolar, oro, bitcoin);       // un solo punto de espera

Console.WriteLine($"Dólar: {await dolar}");    // ya terminaron: el await es inmediato
Console.WriteLine($"Oro: {await oro}");
Console.WriteLine($"Bitcoin: {await bitcoin}");
Console.WriteLine($"Tiempo total: {reloj.Elapsed.TotalSeconds:F1} s");   // ~3.0, no 6.0

static async Task<string> ObtenerDolarAsync() { await Task.Delay(2000); return "$3.85"; }
static async Task<string> ObtenerOroAsync() { await Task.Delay(3000); return "$1,950.00"; }
static async Task<string> ObtenerBitcoinAsync() { await Task.Delay(1000); return "$64,000.00"; }
```

**Qué observar:** el tiempo total es el de la API más lenta. Después de `WhenAll`, leer `.Result` también sería seguro (las tareas ya terminaron), pero `await` es igual de rápido y es la costumbre más segura: si un día alguien borra el `WhenAll`, `.Result` pasaría a bloquear.

</details>

-----

### 4. Excepciones en tareas

**Objetivo:** capturar el error de una operación asíncrona con un `try`/`catch` normal.

**Contexto:** la aplicación se conecta a una pasarela de pagos que está caída.

**Instrucciones:**

1. Crea `ProcesarPagoAsync(decimal monto)` que, tras 1,5 s, lance `HttpRequestException("503 Service Unavailable")`.
2. Llámalo con `await` dentro de un `try`/`catch`.
3. Muestra un mensaje amigable. Después, prueba a mover el `try` para que envuelva solo la **llamada** (guardando la tarea) y no el `await`: ¿qué pasa?

<details>
<summary>Solución</summary>

```csharp
Console.WriteLine("Procesando pago...");

try
{
    await ProcesarPagoAsync(500.50m);
    Console.WriteLine("Pago completado.");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"[Error de red] {ex.Message}");
    Console.WriteLine("Intenta con otra tarjeta o más tarde.");
}

// Variante del paso 3:
Task tarea;
try
{
    tarea = ProcesarPagoAsync(10m);      // aquí NO se lanza nada: la excepción queda guardada en la tarea
}
catch (HttpRequestException)
{
    Console.WriteLine("Esto nunca se imprime");
    throw;
}
try { await tarea; }                     // aparece recién al esperarla
catch (HttpRequestException ex) { Console.WriteLine($"Capturada al hacer await: {ex.Message}"); }

static async Task ProcesarPagoAsync(decimal monto)
{
    Console.WriteLine($"[Pasarela] Enviando {monto:N2}...");
    await Task.Delay(1500);
    throw new HttpRequestException("503 Service Unavailable");
}
```

**Qué observar:** `await` relanza la excepción original (no una `AggregateException`), así que el `catch` funciona como en el código sincrónico. Pero el `try` tiene que envolver el **`await`**, no la llamada. `AggregateException` sigue existiendo: la verías con `.Wait()`, con `.Result` o en `tarea.Exception`.

</details>

-----

### 5. Cancelación por tiempo límite: `CancelAfter`

**Objetivo:** abortar una operación que tarda demasiado.

**Contexto:** un reporte histórico tarda 10 s en generarse; el sistema tiene un límite de 3 s.

**Instrucciones:**

1. Crea un `CancellationTokenSource` y prográmalo con `CancelAfter(3000)`.
2. Pasa el token a `GenerarReporteAsync(CancellationToken token)`, que espera 10 s con `Task.Delay(10000, token)`.
3. Captura la cancelación.

<details>
<summary>Solución</summary>

```csharp
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(3));

try
{
    await GenerarReporteAsync(cts.Token);
    Console.WriteLine("Reporte generado.");
}
catch (OperationCanceledException)          // base de TaskCanceledException: captura ambas
{
    Console.WriteLine("[Abortado] El reporte superó el tiempo límite. Recursos liberados.");
}

static async Task GenerarReporteAsync(CancellationToken token)
{
    Console.WriteLine("[SQL] Procesando millones de registros (10 s)...");
    await Task.Delay(10_000, token);        // escucha el token: se interrumpe al cancelarse
    Console.WriteLine("[SQL] Terminado.");
}
```

**Qué observar:** `Task.Delay` cancelado lanza `TaskCanceledException`, que hereda de `OperationCanceledException`. Captura siempre `OperationCanceledException`: así también atrapas la que lanza `ThrowIfCancellationRequested()` (próximo ejercicio). Si no le pasas el token a la base de datos o al `HttpClient`, la operación sigue trabajando en segundo plano aunque nadie espere el resultado.

</details>

-----

### 6. Cancelación manual: `ThrowIfCancellationRequested`

**Objetivo:** cancelar a pedido un proceso que avanza por pasos.

**Contexto:** se exporta un reporte de 10 páginas (1 s cada una); el usuario se arrepiente a los 2 s.

**Instrucciones:**

1. Crea `ExportarAsync(CancellationToken token)` con un bucle de 10 páginas que, en cada vuelta, llame a `token.ThrowIfCancellationRequested()` y espere 1 s.
2. En el programa principal, inicia la tarea **sin** esperarla, espera 2 s y llama a `cts.Cancel()`.
3. Espera la tarea y captura la cancelación.

<details>
<summary>Solución</summary>

```csharp
using var cts = new CancellationTokenSource();

Task exportacion = ExportarAsync(cts.Token);

await Task.Delay(2000);
Console.WriteLine("[Usuario] Cancelar");
cts.Cancel();

try
{
    await exportacion;
}
catch (OperationCanceledException)
{
    Console.WriteLine("[Sistema] Exportación cancelada de forma segura.");
}

static async Task ExportarAsync(CancellationToken token)
{
    for (int pagina = 1; pagina <= 10; pagina++)
    {
        token.ThrowIfCancellationRequested();     // punto de control seguro
        Console.WriteLine($"Generando página {pagina}/10...");
        await Task.Delay(1000, token);
    }
    Console.WriteLine("Exportación terminada.");
}
```

**Qué observar:** la cancelación es **cooperativa**: nadie "mata" la tarea; el método comprueba el token en puntos donde detenerse es seguro (entre páginas), así no deja archivos a medio escribir.

</details>

-----

### 7. Progreso: `IProgress<T>`

**Objetivo:** informar el avance de una tarea sin acoplarla a la forma de mostrarlo.

**Contexto:** subir un archivo pesado y mostrar "20 %... 40 %... 100 %".

**Instrucciones:**

1. Crea `SubirArchivoAsync(IProgress<int> progreso)` que avance de 20 en 20 con una espera de 800 ms y llame a `progreso.Report(porcentaje)`.
2. Pásale un `Progress<int>` con una lambda que escriba el porcentaje.
3. Ejecuta varias veces: ¿el último porcentaje siempre aparece antes de "Carga completada"? Prueba después con una clase propia que implemente `IProgress<int>`.

<details>
<summary>Solución</summary>

```csharp
Console.WriteLine("Versión con Progress<T>:");
await SubirArchivoAsync(new Progress<int>(p => Console.Write($"\rSubiendo: {p}%   ")));
Console.WriteLine("\nCarga completada (a veces el 100% aparece DESPUÉS de esta línea)");

Console.WriteLine("\nVersión con IProgress<T> propio:");
await SubirArchivoAsync(new ProgresoConsola());
Console.WriteLine("\nCarga completada (siempre en orden)");

static async Task SubirArchivoAsync(IProgress<int>? progreso = null)
{
    for (int porcentaje = 0; porcentaje <= 100; porcentaje += 20)
    {
        await Task.Delay(800);
        progreso?.Report(porcentaje);
    }
}

class ProgresoConsola : IProgress<int>
{
    public void Report(int valor) => Console.Write($"\rSubiendo: {valor}%   ");   // se ejecuta de inmediato, en el mismo hilo
}
```

**Qué observar:** `Progress<T>` envía cada reporte al contexto donde se creó. En WPF o MAUI es el hilo de la interfaz (perfecto para actualizar una barra); en consola no hay contexto, así que los reportes van al thread pool y pueden llegar tarde o desordenados. El método `SubirArchivoAsync` no cambia en ninguna de las dos versiones: solo conoce la interfaz `IProgress<int>`.

</details>

-----

### 8. Carrera de servidores espejo: `Task.WhenAny`

**Objetivo:** quedarse con el primer resultado de varias operaciones equivalentes.

**Contexto:** un archivo está en tres servidores espejo (EE. UU. 2,5 s, Europa 4 s, Asia 1 s); se pide a los tres y se usa el primero que responda.

**Instrucciones:**

1. Lanza los tres métodos sin `await`.
2. `await Task.WhenAny(...)` devuelve la **tarea** ganadora; espérala para obtener su resultado.
3. Muestra cuánto tardó y comprueba si las otras dos siguen ejecutándose.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;

var reloj = Stopwatch.StartNew();

Task<string> usa = DescargarAsync("EE. UU.", 2500);
Task<string> europa = DescargarAsync("Europa", 4000);
Task<string> asia = DescargarAsync("Asia", 1000);

Task<string> ganadora = await Task.WhenAny(usa, europa, asia);
Console.WriteLine($"Descargado desde {await ganadora} en {reloj.Elapsed.TotalSeconds:F1} s");

Console.WriteLine($"¿EE. UU. terminó? {usa.IsCompleted}   ¿Europa terminó? {europa.IsCompleted}");
// False y False: SIGUEN corriendo. WhenAny no cancela nada (ver el reto 3).

static async Task<string> DescargarAsync(string servidor, int ms)
{
    await Task.Delay(ms);
    return servidor;
}
```

**Qué observar:** `WhenAny` devuelve `Task<Task<string>>`; el primer `await` da la tarea ganadora y el segundo, su resultado. Las perdedoras no se detienen solas: en un sistema real siguen consumiendo red hasta que las canceles.

</details>

-----

### 9. Streaming: `IAsyncEnumerable<T>`

**Objetivo:** procesar registros a medida que llegan, sin cargar todos en memoria.

**Contexto:** un reporte de miles de notas de alumnos que se leen de la base de datos por bloques.

**Instrucciones:**

1. Crea `async IAsyncEnumerable<string> ObtenerRegistrosAsync()` que, en un bucle, espere 1 s y haga `yield return` de un registro.
2. Consúmelo con `await foreach`.
3. Agrega un `CancellationToken` y corta el streaming después del tercer registro.

<details>
<summary>Solución</summary>

```csharp
using System.Runtime.CompilerServices;

using var cts = new CancellationTokenSource();
int procesados = 0;

try
{
    await foreach (string registro in ObtenerRegistrosAsync(cts.Token))
    {
        Console.WriteLine($"Procesando: {registro}");
        if (++procesados == 3) cts.Cancel();          // el consumidor decide cortar
    }
}
catch (OperationCanceledException)
{
    Console.WriteLine("Streaming cancelado después de 3 registros.");
}

static async IAsyncEnumerable<string> ObtenerRegistrosAsync(
    [EnumeratorCancellation] CancellationToken token = default)
{
    for (int i = 1; i <= 1000; i++)
    {
        await Task.Delay(1000, token);                // simula leer un bloque de la base de datos
        yield return $"Nota_Alumno_{i}";
    }
}
```

**Qué observar:** en memoria hay un registro por vez, sean 5 o 5 millones. (La versión original tenía `async unsafe`: `unsafe` no tiene nada que ver con la asincronía y hace que el método no compile). `[EnumeratorCancellation]` permite pasar el token también con `ObtenerRegistrosAsync().WithCancellation(token)`.

</details>

-----

## Retos

### Reto 1: Orquestador de datos académicos (`WhenAll` + fallos parciales)

**Misión:**

1. Crea `CargarAlumnosAsync`, `CargarNotasAsync` y `CargarAsistenciasAsync`; cada una tarda entre 1 y 3 segundos (aleatorio).
2. `CargarAsistenciasAsync` lanza una excepción de forma aleatoria (por ejemplo, el 50 % de las veces).
3. Lanza las tres y espéralas con `Task.WhenAll` dentro de un `try`/`catch`.
4. Aunque las asistencias fallen, procesa los alumnos y las notas que sí llegaron.
5. Muestra el tiempo total (no debe superar los ~3 s).

**Pista:** cuando `await Task.WhenAll(...)` lanza la excepción, **las otras tareas ya terminaron**. Revisa cada una con `IsCompletedSuccessfully`.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;

var reloj = Stopwatch.StartNew();

Task<string[]> alumnos = CargarAlumnosAsync();
Task<string[]> notas = CargarNotasAsync();
Task<string[]> asistencias = CargarAsistenciasAsync();
Task todas = Task.WhenAll(alumnos, notas, asistencias);

try
{
    await todas;
}
catch
{
    foreach (var ex in todas.Exception!.InnerExceptions)
        Console.WriteLine($"[Aviso] Una fuente falló: {ex.Message}");
}

Mostrar("Alumnos", alumnos);
Mostrar("Notas", notas);
Mostrar("Asistencias", asistencias);
Console.WriteLine($"Tiempo total: {reloj.Elapsed.TotalSeconds:F1} s");

static void Mostrar(string fuente, Task<string[]> tarea) =>
    Console.WriteLine(tarea.IsCompletedSuccessfully
        ? $"{fuente}: {string.Join(", ", tarea.Result)}"
        : $"{fuente}: no disponible");

static async Task<string[]> CargarAlumnosAsync()
{
    await Task.Delay(Random.Shared.Next(1000, 3001));
    return new[] { "Ana", "Luis", "Eva" };
}

static async Task<string[]> CargarNotasAsync()
{
    await Task.Delay(Random.Shared.Next(1000, 3001));
    return new[] { "Ana: 18", "Luis: 14", "Eva: 16" };
}

static async Task<string[]> CargarAsistenciasAsync()
{
    await Task.Delay(Random.Shared.Next(1000, 3001));
    if (Random.Shared.Next(2) == 0)
        throw new InvalidOperationException("El servicio de asistencias no responde");
    return new[] { "Ana: 95%", "Luis: 88%", "Eva: 100%" };
}
```

`tarea.Result` es seguro dentro de `Mostrar` porque solo se lee cuando `IsCompletedSuccessfully` es `true`. `Random.Shared` evita crear un `Random` en cada llamada.

</details>

-----

### Reto 2: Exportador con progreso y botón de pánico (`IProgress` + cancelación)

**Misión:**

1. Crea `GenerarReporteErpAsync(IProgress<int> progreso, CancellationToken token)` que genere 100 páginas: por cada página espera 100 ms, verifica la cancelación y reporta el porcentaje.
2. En el programa principal, muestra un contador dinámico en la misma línea (`\r`).
3. Cancela automáticamente si tarda más de 5 segundos (`CancelAfter(5000)`). Como el reporte completo tarda ~10 s, debe cancelarse.
4. Bonus: permite cancelar antes presionando una tecla.

<details>
<summary>Solución</summary>

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

// Bonus: cancelar al presionar una tecla (si la consola lo permite)
_ = Task.Run(() =>
{
    Console.ReadKey(intercept: true);
    cts.Cancel();
});

Console.WriteLine("Generando reporte (presiona una tecla para cancelar)...");

try
{
    await GenerarReporteErpAsync(new ProgresoConsola(), cts.Token);
    Console.WriteLine("\nReporte completo.");
}
catch (OperationCanceledException)
{
    Console.WriteLine("\nReporte cancelado (tiempo límite o pedido del usuario).");
}

static async Task GenerarReporteErpAsync(IProgress<int> progreso, CancellationToken token)
{
    const int TotalPaginas = 100;
    for (int pagina = 1; pagina <= TotalPaginas; pagina++)
    {
        token.ThrowIfCancellationRequested();
        await Task.Delay(100, token);
        progreso.Report(pagina * 100 / TotalPaginas);
    }
}

class ProgresoConsola : IProgress<int>
{
    public void Report(int porcentaje) => Console.Write($"\rProcesando: {porcentaje,3}%");
}
```

Se usa una implementación propia de `IProgress<int>` para que, en consola, el contador se escriba en orden (ver el ejercicio 7). La tarea del bonus usa el descarte `_ =` porque no se espera: queda escuchando el teclado.

</details>

-----

### Reto 3: El banco más rápido (`WhenAny` + cancelar a los perdedores)

**Misión:**

1. Crea un método que simule consultar el tipo de cambio en un banco, con un retraso aleatorio y que respete un `CancellationToken`.
2. Consulta tres bancos a la vez y quédate con el primero que responda (`Task.WhenAny`).
3. Cancela las otras dos consultas y confirma que terminaron canceladas.

<details>
<summary>Solución</summary>

```csharp
using var cts = new CancellationTokenSource();

var consultas = new List<Task<string>>
{
    ConsultarBancoAsync("Banco A", cts.Token),
    ConsultarBancoAsync("Banco B", cts.Token),
    ConsultarBancoAsync("Banco C", cts.Token)
};

Task<string> ganadora = await Task.WhenAny(consultas);
Console.WriteLine($"Ganador: {await ganadora}");

cts.Cancel();                                   // ya no necesitamos a los demás

foreach (var tarea in consultas.Where(t => t != ganadora))
{
    try
    {
        await tarea;                            // observar el resultado evita excepciones "perdidas"
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Una consulta perdedora fue cancelada.");
    }
}

static async Task<string> ConsultarBancoAsync(string banco, CancellationToken token)
{
    int demora = Random.Shared.Next(500, 3000);
    await Task.Delay(demora, token);
    return $"{banco}: Q7.{Random.Shared.Next(70, 90)} (en {demora} ms)";
}
```

Es posible (poco probable) que una perdedora termine justo antes del `Cancel()`: en ese caso su `await` devuelve el valor sin excepción, y el código lo tolera.

</details>

-----

## Checkpoint

Al terminar este cuaderno deberías poder explicar:

* Por qué `Task.Delay` y las APIs `...Async` escalan y `Thread.Sleep` no.
* Por qué `await` va dentro del `try`, y cuándo aparece `AggregateException`.
* Por qué `WhenAll` tarda lo que la tarea más lenta y `WhenAny` lo que la más rápida, y por qué hay que cancelar a las perdedoras.
* Qué significa que la cancelación sea cooperativa.
* Cuándo usar `IProgress<T>` y por qué `Progress<T>` se comporta distinto en consola y en una interfaz gráfica.
* Cómo `IAsyncEnumerable<T>` procesa grandes volúmenes con memoria constante.
