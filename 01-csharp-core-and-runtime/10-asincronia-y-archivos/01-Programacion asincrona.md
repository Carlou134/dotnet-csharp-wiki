# Programación asíncrona

## En una frase

Con `async` y `await`, un método puede **esperar una operación lenta** (una llamada a una API, una consulta a la base de datos, leer un archivo) **sin bloquear el hilo** que lo ejecuta; el método devuelve una `Task` (o `Task<T>`) que representa ese trabajo en curso y su resultado futuro.

-----

## Antes de empezar

Conviene que ya sepas:

* Métodos, valores de retorno y genéricos (`Task<T>` es genérico), de [Genéricos](../05-tipos-avanzados/04-Genericos.md).
* Excepciones y `try`/`catch`, de [Manejo de excepciones](../08-excepciones/01-Manejo%20de%20excepciones.md).
* Lambdas, de [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Sincrónico (bloqueante):** cada operación termina antes de que empiece la siguiente; mientras espera, el hilo no hace nada más.
* **Asíncrono:** se inicia una operación y, mientras se completa, el hilo queda libre para otra cosa.
* **Hilo (*thread*):** una secuencia de ejecución; un programa puede tener varios.
* **`Task` / `Task<T>`:** objeto que representa una operación en curso (sin resultado / con resultado de tipo `T`).
* **`async`:** modificador que permite usar `await` dentro de un método.
* **`await`:** espera a que una `Task` termine **sin bloquear** el hilo y obtiene su resultado.
* **Operación de E/S (*I/O-bound*):** espera por algo externo: red, disco, base de datos.
* **Operación de CPU (*CPU-bound*):** cálculo intensivo que mantiene ocupado al procesador.
* **TAP (*Task-based Asynchronous Pattern*):** el patrón de .NET basado en `Task`, `async` y `await`.

-----

## El problema

Una aplicación consulta el clima de tres ciudades a una API externa. Cada consulta tarda 1 segundo:

```csharp
string lima = ObtenerClima("Lima");        // 1 s esperando la red: el hilo no hace NADA
string cusco = ObtenerClima("Cusco");      // otro segundo
string piura = ObtenerClima("Piura");      // otro segundo
// total: 3 segundos
```

Durante esos 3 segundos:

* En una aplicación de escritorio, la ventana se **congela**: no responde a clics.
* En un servidor web, ese hilo queda **bloqueado** esperando, sin poder atender a otros usuarios. Con muchas peticiones simultáneas, el servidor se queda sin hilos y deja de responder.

Y las tres consultas son independientes: no hay motivo para esperar a que termine una para empezar la siguiente.

-----

## Cómo funciona

### `async`, `await` y `Task`

```csharp
static async Task<string> ObtenerClimaAsync(string ciudad)
{
    await Task.Delay(1000);                        // simula 1 s de espera por la red
    return $"{ciudad}: 22 °C";
}
```

* `async` en la firma habilita el uso de `await` dentro del método.
* El tipo de retorno es `Task<string>`: "una operación que, cuando termine, tendrá un `string`". Dentro del método haces `return` de un `string` normal; el compilador lo envuelve en la `Task`.
* Si no devuelve nada, el tipo es `Task` (el equivalente asíncrono de `void`).
* Por convención, el nombre termina en **`Async`**.
* `Task.Delay(1000)` es una espera asíncrona de 1 segundo; la usaremos para simular operaciones lentas.

Para usarlo:

```csharp
string clima = await ObtenerClimaAsync("Lima");    // espera SIN bloquear y obtiene el string
Console.WriteLine(clima);
```

En `Program.cs` con top-level statements puedes usar `await` directamente. Con un `Main` explícito, debe ser `static async Task Main()`.

### Qué hace realmente `await`

```csharp
Console.WriteLine("1. Antes");
string clima = await ObtenerClimaAsync("Lima");
Console.WriteLine("3. Después: " + clima);
```

1. Se llama a `ObtenerClimaAsync`, que empieza a ejecutarse hasta su primer `await` incompleto (`Task.Delay`).
2. Como la tarea no terminó, `await` **devuelve el control** a quien llamó y **libera el hilo**: en una interfaz gráfica, la ventana sigue respondiendo; en un servidor, el hilo atiende otra petición.
3. Cuando la espera termina, el método **continúa** desde la línea siguiente al `await` (posiblemente en otro hilo) y la ejecución sigue hasta el final.

Desde el punto de vista de tu código, se lee como si fuera secuencial. La diferencia es que, mientras espera, **nadie queda bloqueado**.

**La metáfora del restaurante.** Un mesero (el hilo) toma tu pedido y lo lleva a la cocina (la base de datos):

* **Sincrónico:** el mesero se queda parado en la cocina 20 minutos esperando tu plato. Si llegan clientes nuevos, nadie los atiende.
* **Asíncrono:** el mesero deja el pedido (`await`) y vuelve al salón a atender otras mesas. Cuando la cocina toca la campana, **el primer mesero libre** (no necesariamente el mismo) te lleva el plato.

La cocina no cocina más rápido: lo que mejora es cuántos clientes atiende el restaurante con los mismos meseros. Eso es la **escalabilidad**: async no acelera una consulta, permite que el servidor atienda muchas más peticiones a la vez sin quedarse sin hilos (*thread starvation*).

```text
Sincrónico:  [hilo ocupado esperando ████████████] → sigue
Asíncrono:   [inicia] → hilo libre para otras cosas … [la red responde] → continúa
```

### Iniciar sin esperar y esperar después

Llamar a un método `async` **inicia** la operación y devuelve la `Task` inmediatamente. Puedes hacer otras cosas antes de esperarla:

```csharp
Task<string> tarea = ObtenerClimaAsync("Lima");   // empieza YA
Console.WriteLine("Mientras tanto, preparo la pantalla...");
string clima = await tarea;                        // ahora espero su resultado
```

Con eso, las tres consultas del problema pueden correr **a la vez**:

```csharp
Task<string> t1 = ObtenerClimaAsync("Lima");      // las tres empiezan casi al mismo tiempo
Task<string> t2 = ObtenerClimaAsync("Cusco");
Task<string> t3 = ObtenerClimaAsync("Piura");

Console.WriteLine(await t1);
Console.WriteLine(await t2);
Console.WriteLine(await t3);
// total: ~1 segundo en lugar de 3
```

```text
SECUENCIAL (await uno por uno)
tiempo →   0s            1s            2s            3s
Lima       ├─ esperando ─┤
Cusco                    ├─ esperando ─┤
Piura                                  ├─ esperando ─┤      total: 3 s

CONCURRENTE (iniciar todas, esperar después)
tiempo →   0s            1s
Lima       ├─ esperando ─┤
Cusco      ├─ esperando ─┤
Piura      ├─ esperando ─┤                                  total: ~1 s
```

Mientras "espera", ninguna de las tres ocupa un hilo: solo hay una petición en vuelo por la red.

Esperar varias tareas a la vez (`Task.WhenAll`) se ve en la [próxima lección](02-Concurrencia%20cancelacion%20y%20errores.md).

Ojo con el tipo: si guardas la tarea como `Task` (sin `<string>`), `await` no devuelve nada:

```csharp
Task tarea = ObtenerClimaAsync("Lima");
string clima = await tarea;   // error CS0029: no se puede convertir 'void' a 'string'
```

### `Task.Delay` frente a `Thread.Sleep`

```csharp
await Task.Delay(1000);    // espera asíncrona: el hilo queda LIBRE
Thread.Sleep(1000);        // espera sincrónica: el hilo queda BLOQUEADO
```

Dentro de código asíncrono, nunca uses `Thread.Sleep`.

### Asíncrono hasta arriba ("async all the way")

Cuando un método usa `await`, se vuelve `async` y devuelve `Task`, y normalmente quien lo llama también hace `await`, y así hacia arriba hasta el punto de entrada (o el controlador de la API). Esto se llama *async all the way*.

La tentación es "cortar" la cadena bloqueando:

```csharp
string clima = ObtenerClimaAsync("Lima").Result;    // ❌ bloquea el hilo hasta que termine
ObtenerClimaAsync("Lima").Wait();                   // ❌ lo mismo
```

`.Result` y `.Wait()` bloquean el hilo, eliminando todo el beneficio, y en algunos entornos (aplicaciones de escritorio, ASP.NET clásico) provocan un **bloqueo mutuo** (*deadlock*): el hilo espera a la tarea, y la tarea espera a que el hilo quede libre para continuar. El programa se cuelga para siempre.

### `async void`: solo para manejadores de eventos

```csharp
async void Guardar() { await ...; }   // ❌ quien llama no puede esperarlo ni capturar sus excepciones
async Task GuardarAsync() { await ...; }   // ✅
```

Un método `async void` no devuelve una `Task`, así que nadie puede hacer `await` sobre él ni saber cuándo terminó, y si lanza una excepción, puede terminar el proceso. Su único uso legítimo son los manejadores de eventos de interfaces gráficas (`async void Boton_Click(object sender, EventArgs e)`), cuya firma la impone el framework.

### E/S frente a CPU: `Task.Run`

`async`/`await` brilla con operaciones de **E/S**: mientras se espera la red o el disco, no se usa ningún hilo. Las APIs de .NET ya traen versiones asíncronas: `HttpClient.GetStringAsync`, `File.ReadAllTextAsync`, `SaveChangesAsync` de Entity Framework...

Para un **cálculo pesado** (CPU), no hay nada que "esperar": el procesador está trabajando. Si quieres que no bloquee la interfaz gráfica, puedes mandarlo a otro hilo del *thread pool* con `Task.Run`:

```csharp
long resultado = await Task.Run(() => CalcularPrimos(10_000_000));   // el cálculo corre en otro hilo
```

En un servidor web, `Task.Run` para trabajo de CPU no ayuda (solo cambia de hilo); ahí lo que libera recursos es la E/S asíncrona.

-----

## Ejemplo completo

Comparar la versión secuencial con la concurrente:

```csharp
using System.Diagnostics;

string[] ciudades = { "Lima", "Cusco", "Piura" };
var reloj = Stopwatch.StartNew();

// 1. Secuencial: espera cada una antes de empezar la siguiente
foreach (string ciudad in ciudades)
{
    Console.WriteLine(await ObtenerClimaAsync(ciudad));
}
Console.WriteLine($"Secuencial: {reloj.ElapsedMilliseconds} ms\n");

// 2. Concurrente: inicia todas y luego espera los resultados
reloj.Restart();
var tareas = new List<Task<string>>();
foreach (string ciudad in ciudades)
{
    tareas.Add(ObtenerClimaAsync(ciudad));     // inicia sin esperar
}
foreach (Task<string> t in tareas)
{
    Console.WriteLine(await t);                // espera cada resultado
}
Console.WriteLine($"Concurrente: {reloj.ElapsedMilliseconds} ms\n");

// 3. Mientras se espera, el programa puede hacer otra cosa
Task<string> pendiente = ObtenerClimaAsync("Tacna");
for (int i = 1; i <= 3; i++)
{
    Console.WriteLine($"Haciendo otra cosa ({i})...");
    await Task.Delay(200);
}
Console.WriteLine(await pendiente);

static async Task<string> ObtenerClimaAsync(string ciudad)
{
    await Task.Delay(1000);   // simula la latencia de la red
    int temperatura = 15 + ciudad.Length;
    return $"{ciudad}: {temperatura} °C";
}
```

Salida (los milisegundos varían un poco):

```text
Lima: 19 °C
Cusco: 20 °C
Piura: 20 °C
Secuencial: 3021 ms

Lima: 19 °C
Cusco: 20 °C
Piura: 20 °C
Concurrente: 1008 ms

Haciendo otra cosa (1)...
Haciendo otra cosa (2)...
Haciendo otra cosa (3)...
Tacna: 20 °C
```

El mismo trabajo pasa de 3 segundos a 1, solo cambiando **cuándo** se espera.

-----

## Errores comunes

**1. `await` en un método que no es `async`.**
Qué pasa: `error CS4032: The 'await' operator can only be used within an async method. Consider marking this method with the 'async' modifier and changing its return type to 'Task<string>'.`
Por qué: `await` requiere que el compilador transforme el método.
Arreglo: agrega `async` y cambia el retorno a `Task`/`Task<T>`.

**2. Esperar el resultado de una `Task` sin tipo.**
Qué pasa: `error CS0029: Cannot implicitly convert type 'void' to 'string'`.
Por qué: `await` sobre `Task` no devuelve nada; solo `Task<T>` devuelve un `T`.
Arreglo: `Task<string> tarea = ...`.

**3. Olvidar el `await`.**
Qué pasa: `warning CS4014: Because this call is not awaited, execution of the current method continues before the call is completed`. La operación puede no terminar, o sus excepciones se pierden.
Por qué: llamar a un método `async` solo inicia la tarea.
Arreglo: `await` la tarea (en ese momento o más tarde).

**4. Bloquear con `.Result` o `.Wait()`.**
Qué pasa: rendimiento pobre o un bloqueo mutuo (el programa se cuelga).
Por qué: bloquea el hilo que la tarea podría necesitar para continuar.
Arreglo: `await` en toda la cadena.

**5. `async void` fuera de manejadores de eventos.**
Qué pasa: no se puede esperar, y una excepción puede terminar el proceso.
Por qué: no hay `Task` que represente la operación.
Arreglo: `async Task`.

**6. `Thread.Sleep` dentro de código asíncrono.**
Qué pasa: el hilo queda bloqueado y se pierde el beneficio de la asincronía.
Por qué: `Thread.Sleep` es sincrónico.
Arreglo: `await Task.Delay(...)`.

-----

## Según la versión de C#

* **.NET 4.0:** `Task` y la *Task Parallel Library* (sin `async`/`await`; se encadenaban continuaciones a mano con `ContinueWith`).
* **C# 5 (2012):** `async` y `await`.
* **C# 7:** `ValueTask<T>` y tipos de retorno asíncronos personalizados.
* **C# 7.1:** `async Task Main`.
* **C# 8:** flujos asíncronos (`IAsyncEnumerable<T>`, `await foreach`) y `await using`.
* **C# 9:** `await` directamente en top-level statements.

Si ves código con `BeginXxx`/`EndXxx` o `ContinueWith`, es asincronía anterior a `async`/`await`.

-----

## Cuándo sí y cuándo no

**Usa `async`/`await` cuando:**

* Esperas E/S: llamadas HTTP, base de datos, archivos, colas de mensajes. En ASP.NET Core, casi todo el código de acceso a datos debería ser asíncrono.
* Quieres que una interfaz gráfica no se congele.

**No lo uses cuando:**

* El método no espera nada: marcar `async` un método que solo hace cálculos agrega costo sin beneficio.
* El trabajo es de CPU en un servidor: `Task.Run` no da escalabilidad; optimiza el algoritmo o paraleliza con `Parallel`.

-----

## Resumen en 5 líneas

1. `async Task<T> MetodoAsync()` + `await` espera operaciones lentas sin bloquear el hilo.
2. Llamar a un método `async` inicia la tarea; `await` la espera y obtiene su resultado.
3. Inicia varias tareas antes de esperarlas para que corran a la vez.
4. Nunca bloquees con `.Result`/`.Wait()` ni uses `Thread.Sleep`; usa `await` en toda la cadena.
5. `async void` solo para manejadores de eventos; para CPU en la interfaz gráfica, `Task.Run`.

-----

## Para profundizar

<details>
<summary>La máquina de estados de async</summary>

Igual que con los iteradores, el compilador transforma un método `async` en una máquina de estados: las variables locales pasan a ser campos, y cada `await` es un punto donde el método puede "pausarse" y registrar una continuación. Cuando la tarea esperada termina, la continuación reanuda la máquina en el estado siguiente. Si la tarea ya estaba completa al llegar al `await`, el método sigue de largo sin pausarse.

</details>

<details>
<summary>ConfigureAwait(false) y el contexto de sincronización</summary>

En aplicaciones de escritorio, después de un `await` el código vuelve por defecto al hilo de la interfaz gráfica (el "contexto de sincronización"), para poder actualizar controles. En código de librerías que no tocan la interfaz, `await tarea.ConfigureAwait(false)` evita ese salto y reduce el riesgo de bloqueos mutuos. En ASP.NET Core no hay contexto de sincronización, así que no es necesario en el código de la aplicación.

</details>

<details>
<summary>ValueTask</summary>

`ValueTask<T>` es un struct que evita crear un objeto `Task` cuando el resultado suele estar disponible de inmediato (por ejemplo, un dato que casi siempre está en caché). Es una optimización para código de alto rendimiento: un `ValueTask` solo debe esperarse una vez. En el código de aplicación habitual, `Task<T>` es la opción correcta.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La programación asíncrona permite que una operación lenta, como una llamada a una API, no bloquee el programa. En C# se usa `async` en el método, que devuelve `Task` o `Task<T>`, y `await` para esperar el resultado sin bloquear el hilo. Si se inician varias tareas antes de esperarlas, se ejecutan a la vez.

### Respuesta ampliada (semi-senior)

`async`/`await` implementa el patrón TAP: el compilador genera una máquina de estados y cada `await` sobre una tarea incompleta registra una continuación y libera el hilo. En operaciones de E/S no se consume ningún hilo mientras se espera, y eso es lo que da escalabilidad en servidores; para trabajo de CPU, `Task.Run` solo lo traslada al *thread pool*. Bloquear con `.Result`/`.Wait()` desperdicia hilos y puede provocar deadlocks cuando existe un contexto de sincronización; `async void` impide observar la tarea y sus excepciones. La cadena debe ser asíncrona de punta a punta; en librerías se usa `ConfigureAwait(false)`, y `ValueTask` es una optimización para rutas calientes con resultados sincrónicos frecuentes.

### Preguntas frecuentes de seguimiento

**1. ¿`async` crea un hilo nuevo?**
No. `async`/`await` no crea hilos; libera el actual mientras se espera. `Task.Run` sí usa un hilo del *thread pool*.

**2. ¿Por qué no usar `.Result`?**
Porque bloquea el hilo hasta que la tarea termina y puede causar un bloqueo mutuo en entornos con contexto de sincronización.

**3. ¿Cuándo se usa `async void`?**
Solo en manejadores de eventos, cuya firma exige `void`.

-----

## Práctica

**Ejercicio 1.** Escribe un método `DescargarAsync(string archivo, int milisegundos)` que simule una descarga con `Task.Delay` y devuelva un mensaje. Descarga tres archivos de 500, 800 y 300 ms primero en forma secuencial y después concurrente, y mide ambos tiempos.

<details>
<summary>Solución</summary>

```csharp
using System.Diagnostics;

var reloj = Stopwatch.StartNew();
Console.WriteLine(await DescargarAsync("a.zip", 500));
Console.WriteLine(await DescargarAsync("b.zip", 800));
Console.WriteLine(await DescargarAsync("c.zip", 300));
Console.WriteLine($"Secuencial: ~{reloj.ElapsedMilliseconds} ms");   // ~1600

reloj.Restart();
var ta = DescargarAsync("a.zip", 500);
var tb = DescargarAsync("b.zip", 800);
var tc = DescargarAsync("c.zip", 300);
Console.WriteLine(await ta);
Console.WriteLine(await tb);
Console.WriteLine(await tc);
Console.WriteLine($"Concurrente: ~{reloj.ElapsedMilliseconds} ms");  // ~800 (lo que tarda la más lenta)

static async Task<string> DescargarAsync(string archivo, int milisegundos)
{
    await Task.Delay(milisegundos);
    return $"{archivo} descargado";
}
```

</details>

**Ejercicio 2.** Encuentra los problemas de este código:

```csharp
async void ProcesarPedido(int id)
{
    Thread.Sleep(500);
    var datos = ObtenerDatosAsync(id).Result;
    Console.WriteLine(datos);
}
```

<details>
<summary>Solución</summary>

1. `async void`: nadie puede esperarlo ni capturar sus errores. Debe ser `async Task`.
2. `Thread.Sleep`: bloquea el hilo. Debe ser `await Task.Delay(500)`.
3. `.Result`: bloquea y puede causar un deadlock. Debe ser `await ObtenerDatosAsync(id)`.
4. Por convención, el nombre debería terminar en `Async`.

```csharp
async Task ProcesarPedidoAsync(int id)
{
    await Task.Delay(500);
    var datos = await ObtenerDatosAsync(id);
    Console.WriteLine(datos);
}
```

</details>

-----

## Siguiente lección

[Concurrencia, cancelación y errores](02-Concurrencia%20cancelacion%20y%20errores.md)
