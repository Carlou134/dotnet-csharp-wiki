# Publish/Subscribe en memoria

## En una frase

Un **bus de eventos en memoria** implementa el patrón **Publish/Subscribe** dentro de un mismo proceso: los consumidores se suscriben a un **tipo** de evento y el productor publica sin conocerlos; para que sea confiable debe aislar los errores de cada suscriptor, permitir desuscribirse, soportar handlers asíncronos y ser seguro con varios hilos.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un evento y qué desacopla: [Eventos y arquitectura dirigida por eventos](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md).
* Delegados (`Action<T>`, `Func<T, Task>`) y su lista de invocación: [Delegados](../../01-csharp-core-and-runtime/09-delegados-y-eventos/01-Delegados.md).
* `async`/`await`, `Task.WhenAll` y `CancellationToken`: [Concurrencia, cancelación y errores](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/02-Concurrencia%20cancelacion%20y%20errores.md).
* Genéricos y `typeof(T)`: [Genéricos](../../01-csharp-core-and-runtime/05-tipos-avanzados/04-Genericos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Publish/Subscribe (Pub/Sub):** patrón donde los publicadores emiten mensajes a un intermediario y los suscriptores los reciben según su interés, sin conocerse entre sí.
* **Suscripción:** el registro de un handler para un tipo de evento; debe poder cancelarse.
* **Aislamiento de errores:** el fallo de un suscriptor no impide que los demás reciban el evento.
* **Copia al escribir (*copy-on-write*):** en lugar de modificar una colección compartida, se crea una nueva; quien la está recorriendo sigue con la anterior sin problemas.
* **Fuga de memoria por suscripción:** un objeto que nunca se desuscribe queda referenciado por el bus y no se libera.

-----

## El problema

El bus "básico" que aparece en muchos tutoriales:

```csharp
public class EventBus
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();

    public void Subscribe<T>(Action<T> handler)
    {
        if (!_handlers.ContainsKey(typeof(T))) _handlers[typeof(T)] = new List<Delegate>();
        _handlers[typeof(T)].Add(handler);
    }

    public void Publish<T>(T @event)
    {
        if (!_handlers.ContainsKey(typeof(T))) return;
        foreach (var handler in _handlers[typeof(T)]) ((Action<T>)handler)(@event);
    }
}
```

Funciona en la demo. En un sistema real falla de seis formas:

| Situación | Qué pasa |
| --- | --- |
| Un suscriptor lanza una excepción | Los siguientes **no se ejecutan** y la excepción le llega al productor |
| Un handler se suscribe mientras se publica | `InvalidOperationException: Collection was modified` |
| Dos hilos publican y se suscriben a la vez | `Dictionary` y `List` no son seguros con hilos: datos corruptos |
| Un objeto se suscribe y luego deja de usarse | No hay forma de desuscribirse: el bus lo mantiene vivo para siempre |
| Se publica `object e = new PedidoCreado(...)` | `typeof(T)` es `object`: **nadie** lo recibe, en silencio |
| Un handler necesita `await` (email, base de datos) | No hay forma de registrarlo; con `async` lambda en un `Action<T>` se convierte en `async void` |

-----

## Cómo funciona

### 1. Guardar los handlers por tipo, en una forma única

Los handlers se guardan envueltos en una firma común asíncrona, para no tener que adivinar su tipo al publicar:

```csharp
Func<object, CancellationToken, Task> envoltorio = (e, ct) => handler((T)e, ct);
```

Así un handler síncrono (`Action<T>`) y uno asíncrono (`Func<T, CancellationToken, Task>`) se guardan igual, y el bus nunca hace un cast arriesgado del delegado completo. (El ejercicio de clase hacía `.Cast<Func<T, Task>>()` sobre handlers registrados como `Action<T>`: **`InvalidCastException`** en tiempo de ejecución.)

### 2. Suscribirse devuelve un `IDisposable`

```csharp
var suscripcion = bus.Subscribe<PedidoCreado>(e => ...);
// ...
suscripcion.Dispose();   // se desuscribe
```

Es el mismo patrón que usan `IObservable<T>`, `CancellationToken.Register` y `IOptionsMonitor.OnChange`. El que se suscribe es dueño de la suscripción y la libera cuando termina; con `using`, se libera solo.

### 3. Copia al escribir: publicar y suscribirse a la vez sin romper nada

```text
  _manejadores[PedidoCreado] ──► [ h1, h2, h3 ]   ◄── PublishAsync toma ESTE array y lo recorre
                                                                  │
  Subscribe(h4) crea un array NUEVO:                              │ (sigue con el viejo,
  _manejadores[PedidoCreado] ──► [ h1, h2, h3, h4 ]               │  sin "Collection was modified")
```

* Suscribirse o desuscribirse **reemplaza** el array dentro de un `lock`.
* Publicar toma una referencia al array actual (dentro del `lock`) y lo recorre **fuera** del lock: los handlers pueden tardar sin bloquear al resto.
* Los arrays nunca se modifican después de creados, así que recorrerlos es seguro.

Publicar es mucho más frecuente que suscribirse, así que pagar la copia al suscribirse es un buen negocio.

### 4. Aislar los errores de cada suscriptor

```csharp
foreach (var manejador in manejadores)
{
    try { await manejador(evento, ct); }
    catch (Exception ex) when (ex is not OperationCanceledException)
    {
        log($"Error en un suscriptor de {tipo.Name}: {ex.Message}");   // se registra y se sigue
    }
}
```

El suscriptor de puntos caído no debe impedir que se envíe el email. La cancelación sí se propaga: si quien publica cancela, se detiene todo.

### 5. Secuencial o concurrente

| Estrategia | Ventaja | Desventaja |
| --- | --- | --- |
| **Secuencial** (`foreach` + `await`) | Orden predecible, fácil de depurar | El tiempo total es la suma de todos |
| **Concurrente** (`Task.WhenAll`) | El tiempo total es el del más lento | Sin orden; si los handlers comparten estado, condiciones de carrera; con `await Task.WhenAll` solo ves la **primera** excepción |

Si eliges concurrente, envuelve cada handler en su propio `try/catch` antes de pasarlo a `WhenAll`, para no perder errores.

### 6. Publicar por el tipo real del evento

```csharp
var tipo = evento.GetType();   // no typeof(T)
```

Con `typeof(T)`, publicar a través de una variable `object` o de una interfaz `IEvento` encuentra cero suscriptores. Con `GetType()` se usa el tipo real. (Entregar también a suscriptores de tipos base o interfaces es posible, pero agrega complejidad; ver [Para profundizar](#para-profundizar).)

### 7. Bus en memoria frente al `event` de C#

| | `event` de C# | Bus en memoria |
| --- | --- | --- |
| Quién conoce a quién | El suscriptor conoce **al objeto** que publica (`pedido.Creado += ...`) | Nadie se conoce: solo el bus y el tipo del evento |
| Alcance | Un objeto | Toda la aplicación |
| Errores, async, desuscripción | Manuales | Resueltos una vez en el bus |

-----

## Ejemplo completo

Una aplicación de consola (.NET 10):

```csharp
var bus = new EventBus(log: mensaje => Console.WriteLine($"  [Bus] {mensaje}"));

var email = bus.Subscribe<PedidoCreado>(e => Console.WriteLine($"[Email]      Confirmación del pedido {e.PedidoId}"));

bus.Subscribe<PedidoCreado>(async (e, ct) =>
{
    await Task.Delay(50, ct);   // simula una llamada a la base de datos
    Console.WriteLine($"[Inventario] Stock reservado para el pedido {e.PedidoId}");
});

bus.Subscribe<PedidoCreado>(e => throw new InvalidOperationException("Servicio de puntos caído"));

bus.Subscribe<PedidoCreado>(e => Console.WriteLine($"[Analítica]  Venta de {e.Total:N2}"));

await bus.PublishAsync(new PedidoCreado(Guid.NewGuid(), 1, 250m));

Console.WriteLine("--- el email se desuscribe ---");
email.Dispose();
await bus.PublishAsync(new PedidoCreado(Guid.NewGuid(), 2, 80m));

Console.WriteLine("--- publicado como object ---");
object comoObject = new PedidoCreado(Guid.NewGuid(), 3, 40m);
await bus.PublishAsync(comoObject);

Console.WriteLine("--- un evento sin suscriptores ---");
await bus.PublishAsync(new UsuarioRegistrado(Guid.NewGuid(), "ana@correo.com"));

public record PedidoCreado(Guid EventoId, int PedidoId, decimal Total);
public record UsuarioRegistrado(Guid EventoId, string Email);

public sealed class EventBus(Action<string> log)
{
    private readonly Lock _lock = new();
    private readonly Dictionary<Type, Func<object, CancellationToken, Task>[]> _manejadores = new();

    public IDisposable Subscribe<T>(Func<T, CancellationToken, Task> handler) where T : notnull
    {
        Func<object, CancellationToken, Task> envoltorio = (e, ct) => handler((T)e, ct);
        lock (_lock)
            _manejadores[typeof(T)] = [.. Obtener(typeof(T)), envoltorio];   // array NUEVO

        return new Suscripcion(() => Quitar(typeof(T), envoltorio));
    }

    public IDisposable Subscribe<T>(Action<T> handler) where T : notnull =>
        Subscribe<T>((e, _) =>
        {
            handler(e);
            return Task.CompletedTask;
        });

    public async Task PublishAsync<T>(T evento, CancellationToken ct = default) where T : notnull
    {
        var tipo = evento.GetType();
        Func<object, CancellationToken, Task>[] manejadores;
        lock (_lock)
            manejadores = Obtener(tipo);   // el array actual; nadie lo modificará

        if (manejadores.Length == 0)
        {
            log($"{tipo.Name} publicado sin suscriptores");
            return;
        }

        log($"{tipo.Name} → {manejadores.Length} suscriptor(es)");
        foreach (var manejador in manejadores)
        {
            ct.ThrowIfCancellationRequested();
            try
            {
                await manejador(evento, ct);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                log($"Error en un suscriptor de {tipo.Name}: {ex.Message}");
            }
        }
    }

    private Func<object, CancellationToken, Task>[] Obtener(Type tipo) =>
        _manejadores.TryGetValue(tipo, out var actuales) ? actuales : [];

    private void Quitar(Type tipo, Func<object, CancellationToken, Task> envoltorio)
    {
        lock (_lock)
            _manejadores[tipo] = Obtener(tipo).Where(m => !ReferenceEquals(m, envoltorio)).ToArray();
    }

    private sealed class Suscripcion(Action quitar) : IDisposable
    {
        private Action? _quitar = quitar;

        // Interlocked: aunque se llame dos veces (o desde dos hilos), se desuscribe una sola vez.
        public void Dispose() => Interlocked.Exchange(ref _quitar, null)?.Invoke();
    }
}
```

Salida (con cultura `en-US`):

```text
  [Bus] PedidoCreado → 4 suscriptor(es)
[Email]      Confirmación del pedido 1
[Inventario] Stock reservado para el pedido 1
  [Bus] Error en un suscriptor de PedidoCreado: Servicio de puntos caído
[Analítica]  Venta de 250.00
--- el email se desuscribe ---
  [Bus] PedidoCreado → 3 suscriptor(es)
[Inventario] Stock reservado para el pedido 2
  [Bus] Error en un suscriptor de PedidoCreado: Servicio de puntos caído
[Analítica]  Venta de 80.00
--- publicado como object ---
  [Bus] PedidoCreado → 3 suscriptor(es)
[Inventario] Stock reservado para el pedido 3
  [Bus] Error en un suscriptor de PedidoCreado: Servicio de puntos caído
[Analítica]  Venta de 40.00
--- un evento sin suscriptores ---
  [Bus] UsuarioRegistrado publicado sin suscriptores
```

Lo que este bus **todavía no resuelve**: todo ocurre en el mismo proceso y mientras el productor espera. Si la aplicación se cae a mitad de la publicación, los eventos pendientes se pierden. Eso es lo que resuelve un broker: [Entrega confiable e idempotencia](03-Entrega%20confiable%20e%20idempotencia.md).

-----

## Errores comunes

**1. Cast del delegado completo (`(Func<T, Task>)handler`).**
Qué pasa: `InvalidCastException` si el handler se registró con otra firma.
Por qué: `Action<T>` y `Func<T, Task>` son tipos de delegado distintos, no hay conversión entre ellos.
Arreglo: envolver cada handler en una firma común al suscribirlo.

**2. Lambda `async` en un `Action<T>`.**
Qué pasa: `bus.Subscribe<PedidoCreado>(async e => await ...)` compila como `async void`: el bus no espera al handler y sus excepciones pueden tumbar el proceso.
Por qué: una lambda `async` sin valor de retorno convertida a `Action<T>` es `async void`.
Arreglo: una sobrecarga `Subscribe<T>(Func<T, CancellationToken, Task>)` y usarla siempre que haya `await`.

**3. Un suscriptor que falla corta a los demás.**
Qué pasa: el email no se envía porque falló la analítica.
Por qué: la excepción sale del `foreach`.
Arreglo: `try/catch` por handler, registrar el error y continuar.

**4. Modificar la colección mientras se recorre.**
Qué pasa: `InvalidOperationException: Collection was modified`.
Por qué: un handler (o otro hilo) se suscribe durante la publicación.
Arreglo: copia al escribir (arrays inmutables reemplazados bajo `lock`), o `ImmutableArray<T>`.

**5. Suscripciones que nunca se liberan.**
Qué pasa: objetos que deberían morir siguen vivos (y siguen reaccionando a eventos).
Por qué: el bus guarda una referencia al delegado, y el delegado al objeto.
Arreglo: `Subscribe` devuelve `IDisposable`; el suscriptor lo libera (`using`, o en su propio `Dispose`).

**6. Publicar antes de suscribir.**
Qué pasa: el evento se pierde en silencio.
Por qué: un bus en memoria no guarda eventos; solo entrega a quien está suscrito **en ese momento**.
Arreglo: registrar las suscripciones al iniciar la aplicación (o con DI), y registrar en el log los eventos sin suscriptores.

-----

## Según la versión de C#

* **C# 2:** genéricos y métodos anónimos, base para `Subscribe<T>(Action<T>)`.
* **C# 5:** `async`/`await`.
* **C# 12:** constructores primarios y expresiones de colección (`[.. actuales, nuevo]`, `[]`).
* **C# 13 / .NET 9:** `System.Threading.Lock`, un tipo dedicado para `lock`, más claro y eficiente que bloquear sobre un `object`.

-----

## Cuándo sí y cuándo no

**Un bus en memoria encaja cuando:**

* Quieres desacoplar módulos **dentro** de una misma aplicación (un monolito modular).
* Manejas eventos de dominio que pueden perderse si el proceso se cae, o que se procesan dentro de la misma transacción.

**No alcanza cuando:**

* Los consumidores están en **otros procesos o servicios**.
* Perder un evento es inaceptable (cobros, facturas): necesitas un broker persistente y el patrón outbox.
* Necesitas reintentos, colas de mensajes fallidos o escalar consumidores: eso lo dan un broker y librerías como MassTransit, Wolverine o NServiceBus.

-----

## Resumen en 5 líneas

1. Pub/Sub: los suscriptores se registran por tipo de evento y el publicador no los conoce.
2. Envuelve cada handler en una firma asíncrona común; nunca castees delegados de distinto tipo.
3. Aísla los errores de cada suscriptor y propaga solo la cancelación.
4. Copia al escribir + `lock` para ser seguro con hilos; `Subscribe` devuelve `IDisposable`.
5. Un bus en memoria pierde eventos si el proceso se cae: para eso existen los brokers.

-----

## Para profundizar

<details>
<summary>Handlers registrados con inyección de dependencias</summary>

En una aplicación ASP.NET Core, en lugar de lambdas se suelen usar clases:

```csharp
public interface IEventHandler<in T>
{
    Task HandleAsync(T evento, CancellationToken ct);
}

public sealed class EnviarEmailAlCrearPedido(IEmailSender email) : IEventHandler<PedidoCreado>
{
    public Task HandleAsync(PedidoCreado e, CancellationToken ct) => email.EnviarConfirmacionAsync(e.PedidoId, ct);
}

builder.Services.AddScoped<IEventHandler<PedidoCreado>, EnviarEmailAlCrearPedido>();
builder.Services.AddScoped<IEventHandler<PedidoCreado>, ReservarStockAlCrearPedido>();
```

El "bus" pide al contenedor `IEnumerable<IEventHandler<T>>` y los ejecuta. No hace falta suscribirse a mano, cada handler puede tener dependencias *scoped* y agregar uno es agregar una clase. Es lo que hacen las notificaciones de MediatR (que desde 2025 tiene licencia comercial para muchos usos) o Wolverine.

</details>

<details>
<summary>Entregar también a suscriptores de tipos base</summary>

Si alguien se suscribe a `IEventoDePedido` y se publica `PedidoCreado` (que la implementa), se puede recorrer `tipo`, `tipo.BaseType` y `tipo.GetInterfaces()` y juntar los handlers de todos. Es más flexible, pero el orden de entrega y la duplicación (un handler suscrito a dos tipos de la jerarquía) se vuelven más difíciles de razonar. Para la mayoría de los casos, suscribirse al tipo concreto es suficiente.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un bus de eventos en memoria guarda, para cada tipo de evento, una lista de funciones suscritas. Cuando alguien publica un evento, el bus llama a todas las funciones de ese tipo. Así el que publica no conoce a los que reaccionan. Hay que cuidar que si un suscriptor falla, los demás igual se ejecuten.

### Respuesta ampliada (semi-senior)

Implemento el bus con un diccionario de tipo a arrays inmutables de handlers envueltos en una firma asíncrona común: suscribirse reemplaza el array bajo un `lock` (copia al escribir) y publicar recorre una instantánea fuera del lock, así es seguro con hilos y con suscripciones durante la publicación. `Subscribe` devuelve un `IDisposable` para evitar fugas. Aíslo los errores por handler, propago la cancelación, publico por `GetType()` y elijo ejecución secuencial o concurrente según el orden y el estado compartido. En ASP.NET Core prefiero handlers como clases resueltas por DI. Y tengo claro el límite: un bus en memoria no sobrevive a una caída del proceso; para eventos que no pueden perderse uso un broker con outbox.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `Subscribe` devuelve `IDisposable`?**
Para que el suscriptor pueda desuscribirse y no quede referenciado por el bus para siempre.

**2. ¿Qué pasa si registras un handler `async` como `Action<T>`?**
Se convierte en `async void`: el bus no lo espera y sus excepciones no se pueden capturar normalmente.

**3. ¿Ejecutarías los handlers en paralelo?**
Depende: en paralelo baja la latencia, pero se pierde el orden y hay que capturar errores por handler; si comparten estado, secuencial.

-----

## Práctica

**Ejercicio 1.** ¿Qué imprime este código con el bus del problema (el básico)? ¿Y con el de la lección?

```csharp
bus.Subscribe<PedidoCreado>(e =>
{
    Console.WriteLine("A");
    bus.Subscribe<PedidoCreado>(_ => Console.WriteLine("C"));   // se suscribe durante la publicación
});
bus.Subscribe<PedidoCreado>(_ => Console.WriteLine("B"));
bus.Publish(new PedidoCreado(Guid.NewGuid(), 1, 10m));   // en el de la lección: await bus.PublishAsync(...)
```

<details>
<summary>Solución</summary>

* **Bus básico:** imprime `A` y luego lanza `InvalidOperationException: Collection was modified`, porque la lista se modificó mientras el `foreach` la recorría. `B` nunca se imprime.
* **Bus de la lección:** imprime `A` y `B`. La suscripción de `C` crea un array nuevo, pero la publicación sigue con el anterior. En la **siguiente** publicación, `C` sí recibirá el evento (y se agregará otro `C` cada vez que se ejecute `A`, que es un bug del handler, no del bus).

</details>

**Ejercicio 2.** Agrega al bus de la lección un método `PublishConcurrentAsync` que ejecute todos los handlers a la vez con `Task.WhenAll`, sin perder ningún error.

<details>
<summary>Solución</summary>

```csharp
public async Task PublishConcurrentAsync<T>(T evento, CancellationToken ct = default) where T : notnull
{
    var tipo = evento.GetType();
    Func<object, CancellationToken, Task>[] manejadores;
    lock (_lock)
        manejadores = Obtener(tipo);

    await Task.WhenAll(manejadores.Select(Ejecutar));

    async Task Ejecutar(Func<object, CancellationToken, Task> manejador)
    {
        try
        {
            await manejador(evento, ct);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            log($"Error en un suscriptor de {tipo.Name}: {ex.Message}");
        }
    }
}
```

Cada handler captura su propio error, así que `WhenAll` nunca falla por ellos y todos se registran. El orden de la salida ya no es predecible.

</details>

-----

## Siguiente lección

[Entrega confiable e idempotencia](03-Entrega%20confiable%20e%20idempotencia.md)
