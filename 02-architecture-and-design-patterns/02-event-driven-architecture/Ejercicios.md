# Ejercicios de eventos y Publish/Subscribe

Cuaderno de práctica basado en los ejercicios de la Sesión 11, **revisados y corregidos**. Mantienen los nombres en inglés de la clase (`EventBus`, `OrderCreatedEvent`). Cada solución es una aplicación de consola completa (.NET 10, *top-level statements*).

Requisitos previos: las tres lecciones del [módulo](README.md).

-----

## Ejercicios guiados

### Ejercicio 1: un event bus en memoria

**Objetivo:** un bus con varios suscriptores por tipo de evento, que no se rompa en los casos habituales.

**Contexto:** el bus de la guía (`Dictionary<Type, List<Delegate>>`) funciona en la demo, pero:

* si un handler lanza una excepción, los siguientes no se ejecutan;
* si alguien se suscribe durante una publicación: `InvalidOperationException: Collection was modified`;
* no es seguro con varios hilos;
* no hay forma de desuscribirse;
* publica por `typeof(T)`: un evento publicado como `object` no le llega a nadie.

**Instrucciones:**

1. `Subscribe<T>(Action<T>)` devuelve un `IDisposable` para desuscribirse.
2. Guarda arrays inmutables reemplazados bajo `lock` (copia al escribir).
3. Aísla los errores de cada handler y publica por el tipo real (`GetType()`).

<details>
<summary>Solución</summary>

```csharp
var bus = new EventBus();

var first = bus.Subscribe<OrderCreatedEvent>(e => Console.WriteLine($"Handler 1: orden {e.OrderId}"));
bus.Subscribe<OrderCreatedEvent>(_ => throw new InvalidOperationException("falla el handler 2"));
bus.Subscribe<OrderCreatedEvent>(e => Console.WriteLine($"Handler 3: orden {e.OrderId}"));

bus.Publish(new OrderCreatedEvent(1, 100m));

first.Dispose();
object asObject = new OrderCreatedEvent(2, 50m);
bus.Publish(asObject);

public record OrderCreatedEvent(int OrderId, decimal Total);

public sealed class EventBus
{
    private readonly Lock _lock = new();
    private readonly Dictionary<Type, Action<object>[]> _handlers = new();

    public IDisposable Subscribe<T>(Action<T> handler) where T : notnull
    {
        Action<object> wrapper = e => handler((T)e);
        lock (_lock)
            _handlers[typeof(T)] = [.. Get(typeof(T)), wrapper];

        return new Subscription(() =>
        {
            lock (_lock)
                _handlers[typeof(T)] = Get(typeof(T)).Where(h => !ReferenceEquals(h, wrapper)).ToArray();
        });
    }

    public void Publish<T>(T @event) where T : notnull
    {
        var type = @event.GetType();
        Action<object>[] handlers;
        lock (_lock)
            handlers = Get(type);

        if (handlers.Length == 0)
            Console.WriteLine($"[Bus] {type.Name} publicado sin suscriptores");

        foreach (var handler in handlers)
        {
            try
            {
                handler(@event);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[Bus] Error en un handler de {type.Name}: {ex.Message}");
            }
        }
    }

    private Action<object>[] Get(Type type) => _handlers.TryGetValue(type, out var handlers) ? handlers : [];

    private sealed class Subscription(Action unsubscribe) : IDisposable
    {
        private Action? _unsubscribe = unsubscribe;
        public void Dispose() => Interlocked.Exchange(ref _unsubscribe, null)?.Invoke();
    }
}
```

```text
Handler 1: orden 1
[Bus] Error en un handler de OrderCreatedEvent: falla el handler 2
Handler 3: orden 1
[Bus] Error en un handler de OrderCreatedEvent: falla el handler 2
Handler 3: orden 2
```

</details>

**Qué observar:**

* El handler 2 falla, pero el 3 se ejecuta igual.
* Después de `first.Dispose()`, el handler 1 ya no recibe nada.
* El evento publicado como `object` llegó a los suscriptores de `OrderCreatedEvent` gracias a `GetType()`.
* Este bus es síncrono; el ejercicio 5 lo hace asíncrono.

-----

### Ejercicio 2: publicar desde un servicio

**Objetivo:** que el productor dependa solo de una abstracción para publicar.

**Contexto:** en la guía, el evento se publica **antes** de que haya suscriptores, así que **se pierde en silencio**: un bus en memoria no guarda eventos, solo entrega a quien está suscrito en ese momento.

**Instrucciones:**

1. Define `IEventPublisher` con `Publish<T>` e implementala en el `EventBus` del ejercicio 1.
2. `OrderService` recibe un `IEventPublisher`, no el bus concreto ni los consumidores.
3. Comprueba qué pasa al publicar antes y después de suscribirse.

<details>
<summary>Solución</summary>

Agrega al ejercicio 1:

```csharp
public interface IEventPublisher
{
    void Publish<T>(T @event) where T : notnull;
}

// y en la declaración del bus:
public sealed class EventBus : IEventPublisher { /* igual que antes */ }

public class OrderService(IEventPublisher events)
{
    public void Create(int orderId, decimal total)
    {
        Console.WriteLine($"[Orders] Orden {orderId} guardada");
        events.Publish(new OrderCreatedEvent(orderId, total));
    }
}
```

Y el programa:

```csharp
var bus = new EventBus();
var orders = new OrderService(bus);

orders.Create(1, 100m);   // nadie suscrito todavía

bus.Subscribe<OrderCreatedEvent>(e => Console.WriteLine($"[Email] Confirmación de la orden {e.OrderId}"));
orders.Create(2, 250m);
```

```text
[Orders] Orden 1 guardada
[Bus] OrderCreatedEvent publicado sin suscriptores
[Orders] Orden 2 guardada
[Email] Confirmación de la orden 2
```

</details>

**Qué observar:**

* La orden 1 nunca tendrá email. En una aplicación real, las suscripciones se registran **al iniciar** (o se resuelven por DI), antes de atender peticiones.
* `OrderService` no conoce al bus concreto ni a los consumidores: depende de `IEventPublisher` (inversión de dependencias). En una prueba se reemplaza por un publicador falso que guarda los eventos en una lista.

-----

### Ejercicio 3: varios consumidores para el mismo evento

**Objetivo:** que un evento dispare varias reacciones independientes, y que cada consumidor controle su suscripción.

**Instrucciones:**

1. Suscribe email e inventario a `OrderCreatedEvent`.
2. Haz que el inventario sea una clase que se suscribe en su constructor y se desuscribe en `Dispose`.

<details>
<summary>Solución</summary>

```csharp
var bus = new EventBus();

bus.Subscribe<OrderCreatedEvent>(e => Console.WriteLine($"[Email] Enviado para la orden {e.OrderId}"));

using (var inventory = new InventoryModule(bus))
{
    bus.Publish(new OrderCreatedEvent(1, 100m));
}   // aquí el inventario se desuscribe

bus.Publish(new OrderCreatedEvent(2, 80m));

public sealed class InventoryModule : IDisposable
{
    private readonly IDisposable _subscription;

    public InventoryModule(EventBus bus) =>
        _subscription = bus.Subscribe<OrderCreatedEvent>(OnOrderCreated);

    private void OnOrderCreated(OrderCreatedEvent e) =>
        Console.WriteLine($"[Inventario] Actualizado para la orden {e.OrderId}");

    public void Dispose() => _subscription.Dispose();
}

// + OrderCreatedEvent y EventBus del ejercicio 1
```

```text
[Email] Enviado para la orden 1
[Inventario] Actualizado para la orden 1
[Email] Enviado para la orden 2
```

</details>

**Qué observar:** sin desuscripción, `InventoryModule` seguiría vivo (el bus guarda una referencia a `OnOrderCreated`, y con ella a la instancia) y seguiría reaccionando aunque ya no se use: una fuga de memoria y un bug de comportamiento.

-----

### Ejercicio 4: flujo completo

**Objetivo:** modelar un flujo de negocio con eventos, en el orden correcto.

**Contexto:** la guía suscribía "procesar pago", "generar factura" y "enviar notificación" al **mismo** `OrderCreatedEvent`. Problemas:

* La factura se genera **aunque el pago falle**: las tres reacciones son independientes y ninguna sabe si las otras salieron bien.
* Si el handler de pagos lanza una excepción, con el bus de la guía la factura y la notificación ni siquiera se ejecutan.
* "Procesando pago" es más bien un paso que **debe** ocurrir antes que los otros.

**Instrucciones:** encadena los eventos (coreografía):

1. `OrderCreatedEvent` → Pagos cobra y publica `PaymentProcessedEvent` o `PaymentFailedEvent`.
2. `PaymentProcessedEvent` → Facturación y Notificaciones.
3. `PaymentFailedEvent` → Notificaciones avisa del rechazo.

<details>
<summary>Solución</summary>

```csharp
var bus = new EventBus();

// Pagos
bus.Subscribe<OrderCreatedEvent>(e =>
{
    Console.WriteLine($"[Pagos] Cobrando {e.Total:N2} de la orden {e.OrderId}");
    if (e.Total > 1000m)
        bus.Publish(new PaymentFailedEvent(e.OrderId, "límite de la tarjeta excedido"));
    else
        bus.Publish(new PaymentProcessedEvent(e.OrderId, e.Total));
});

// Facturación
bus.Subscribe<PaymentProcessedEvent>(e =>
    Console.WriteLine($"[Facturación] Factura de la orden {e.OrderId} por {e.Amount:N2}"));

// Notificaciones
bus.Subscribe<PaymentProcessedEvent>(e =>
    Console.WriteLine($"[Notificaciones] Pago confirmado de la orden {e.OrderId}"));
bus.Subscribe<PaymentFailedEvent>(e =>
    Console.WriteLine($"[Notificaciones] Pago rechazado de la orden {e.OrderId}: {e.Reason}"));

bus.Publish(new OrderCreatedEvent(10, 250m));
Console.WriteLine("---");
bus.Publish(new OrderCreatedEvent(11, 5000m));

public record PaymentProcessedEvent(int OrderId, decimal Amount);
public record PaymentFailedEvent(int OrderId, string Reason);

// + OrderCreatedEvent y EventBus del ejercicio 1
```

Salida (con cultura `en-US`):

```text
[Pagos] Cobrando 250.00 de la orden 10
[Facturación] Factura de la orden 10 por 250.00
[Notificaciones] Pago confirmado de la orden 10
---
[Pagos] Cobrando 5,000.00 de la orden 11
[Notificaciones] Pago rechazado de la orden 11: límite de la tarjeta excedido
```

</details>

**Qué observar:**

* Cada evento describe un hecho **ya confirmado**: la factura reacciona a "pago realizado", no a "orden creada".
* La orden 11 no se factura. Con el diseño de la guía, se habría facturado.
* Con un bus síncrono, publicar dentro de un handler procesa el evento nuevo **antes** de volver: el flujo es en profundidad. Con un broker real, cada evento se procesaría por separado y en otro momento.
* La guía describía esto como "pipeline sin dependencias". Las dependencias siguen existiendo (Facturación depende de que Pagos publique `PaymentProcessedEvent`), pero ahora están en los **contratos de los eventos**, no en llamadas directas.

-----

### Ejercicio 5: handlers asíncronos

**Objetivo:** soportar handlers con `await` sin romper el bus.

**Contexto:** el `PublishAsync` de la guía hacía `.Cast<Func<T, Task>>()` sobre handlers registrados como `Action<T>`: **`InvalidCastException`** al ejecutarse, porque son tipos de delegado distintos. Además, no había forma de registrar un handler asíncrono, y con `await Task.WhenAll(...)` solo se ve la **primera** excepción.

**Instrucciones:**

1. Guarda todos los handlers como `Func<object, CancellationToken, Task>`.
2. `Subscribe<T>(Func<T, CancellationToken, Task>)` para asíncronos y `Subscribe<T>(Action<T>)` para síncronos.
3. `PublishAsync` ejecuta los handlers concurrentemente, sin perder ningún error.

<details>
<summary>Solución</summary>

```csharp
var bus = new AsyncEventBus();

bus.Subscribe<OrderCreatedEvent>(async (e, ct) =>
{
    await Task.Delay(300, ct);
    Console.WriteLine($"[Email] Enviado para la orden {e.OrderId}");
});
bus.Subscribe<OrderCreatedEvent>(async (e, ct) =>
{
    await Task.Delay(100, ct);
    throw new TimeoutException("el inventario no respondió");
});
bus.Subscribe<OrderCreatedEvent>(e => Console.WriteLine($"[Log] Orden {e.OrderId} creada"));

var start = System.Diagnostics.Stopwatch.GetTimestamp();
await bus.PublishAsync(new OrderCreatedEvent(1, 100m));
Console.WriteLine($"Tiempo total: ~{System.Diagnostics.Stopwatch.GetElapsedTime(start).TotalMilliseconds:N0} ms");

public record OrderCreatedEvent(int OrderId, decimal Total);

public sealed class AsyncEventBus
{
    private readonly Lock _lock = new();
    private readonly Dictionary<Type, Func<object, CancellationToken, Task>[]> _handlers = new();

    public IDisposable Subscribe<T>(Func<T, CancellationToken, Task> handler) where T : notnull
    {
        Func<object, CancellationToken, Task> wrapper = (e, ct) => handler((T)e, ct);
        lock (_lock)
            _handlers[typeof(T)] = [.. Get(typeof(T)), wrapper];

        return new Subscription(() =>
        {
            lock (_lock)
                _handlers[typeof(T)] = Get(typeof(T)).Where(h => !ReferenceEquals(h, wrapper)).ToArray();
        });
    }

    public IDisposable Subscribe<T>(Action<T> handler) where T : notnull =>
        Subscribe<T>((e, _) =>
        {
            handler(e);
            return Task.CompletedTask;
        });

    public async Task PublishAsync<T>(T @event, CancellationToken ct = default) where T : notnull
    {
        var type = @event.GetType();
        Func<object, CancellationToken, Task>[] handlers;
        lock (_lock)
            handlers = Get(type);

        await Task.WhenAll(handlers.Select(Run));

        async Task Run(Func<object, CancellationToken, Task> handler)
        {
            try
            {
                await handler(@event, ct);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                Console.WriteLine($"[Bus] Error en un handler de {type.Name}: {ex.Message}");
            }
        }
    }

    private Func<object, CancellationToken, Task>[] Get(Type type) =>
        _handlers.TryGetValue(type, out var handlers) ? handlers : [];

    private sealed class Subscription(Action unsubscribe) : IDisposable
    {
        private Action? _unsubscribe = unsubscribe;
        public void Dispose() => Interlocked.Exchange(ref _unsubscribe, null)?.Invoke();
    }
}
```

```text
[Log] Orden 1 creada
[Bus] Error en un handler de OrderCreatedEvent: el inventario no respondió
[Email] Enviado para la orden 1
Tiempo total: ~300 ms
```

</details>

**Qué observar:**

* El tiempo total es el del handler más lento (~300 ms), no la suma (~400 ms): se ejecutan a la vez.
* El orden de la salida lo deciden los tiempos de cada handler, no el orden de suscripción.
* Cada handler captura su propio error, así que `WhenAll` no falla y ningún error se pierde.
* Una lambda `async e => ...` pasada a `Subscribe<T>(Action<T>)` sería `async void`: por eso los asíncronos usan la sobrecarga con `Func<..., Task>` (dos parámetros: evento y token).

-----

## Retos

### Reto 1: notificaciones al registrar un usuario

**Misión:** publicar `UserRegisteredEvent` desde un `UserService` y reaccionar con tres handlers: email de bienvenida (asíncrono), registro en el log y cupón de bienvenida. El `UserService` no debe conocer a ninguno.

**Pista:** usa el `AsyncEventBus` del ejercicio 5 y una interfaz para publicar.

<details>
<summary>Solución</summary>

```csharp
var bus = new AsyncEventBus();

bus.Subscribe<UserRegisteredEvent>(async (e, ct) =>
{
    await Task.Delay(100, ct);   // simula el proveedor de email
    Console.WriteLine($"[Email] Bienvenida enviada a {e.Email}");
});
bus.Subscribe<UserRegisteredEvent>(e => Console.WriteLine($"[Log] Usuario {e.UserId} registrado el {e.OccurredAt:yyyy-MM-dd}"));
bus.Subscribe<UserRegisteredEvent>(e => Console.WriteLine($"[Cupones] Cupón BIENVENIDA10 para el usuario {e.UserId}"));

var users = new UserService(bus);
await users.RegisterAsync("ana@correo.com");

public record UserRegisteredEvent(Guid EventId, DateTimeOffset OccurredAt, Guid UserId, string Email);

public class UserService(AsyncEventBus events)
{
    public async Task RegisterAsync(string email)
    {
        var userId = Guid.NewGuid();
        Console.WriteLine($"[Users] Usuario {userId} guardado");
        await events.PublishAsync(new UserRegisteredEvent(Guid.NewGuid(), DateTimeOffset.UtcNow, userId, email));
    }
}

// + AsyncEventBus del ejercicio 5
```

El `EventId` y `OccurredAt` no se usan todavía, pero son los que permitirían detectar duplicados y ordenar eventos si mañana viajan por un broker.

</details>

### Reto 2: integración entre módulos con compensación

**Misión:** tres módulos (Pedidos, Inventario, Pagos) que se comunican solo por eventos:

1. Pedidos publica `OrderPlacedEvent`.
2. Inventario reserva stock y publica `StockReservedEvent` (o `StockUnavailableEvent`).
3. Pagos cobra al recibir `StockReservedEvent` y publica `PaymentProcessedEvent` o `PaymentFailedEvent`.
4. Si el pago falla, Inventario **libera** el stock (compensación).

**Pista:** cada módulo es una clase con un método `Register(AsyncEventBus bus)`; ninguno referencia a los otros.

<details>
<summary>Solución</summary>

```csharp
var bus = new AsyncEventBus();
var inventory = new InventoryModule(stock: new() { ["keyboard"] = 5 });
inventory.Register(bus);
new PaymentsModule().Register(bus);
bus.Subscribe<PaymentProcessedEvent>(e => Console.WriteLine($"[Pedidos] Orden {e.OrderId} confirmada"));
bus.Subscribe<StockUnavailableEvent>(e => Console.WriteLine($"[Pedidos] Orden {e.OrderId} rechazada: sin stock"));

await bus.PublishAsync(new OrderPlacedEvent(1, "keyboard", 2, 300m));
Console.WriteLine("---");
await bus.PublishAsync(new OrderPlacedEvent(2, "keyboard", 1, 9000m));   // el pago fallará
Console.WriteLine("---");
await bus.PublishAsync(new OrderPlacedEvent(3, "keyboard", 10, 1500m));  // no hay stock
Console.WriteLine($"Stock final de keyboard: {inventory.Available("keyboard")}");

public record OrderPlacedEvent(int OrderId, string Sku, int Quantity, decimal Total);
public record StockReservedEvent(int OrderId, string Sku, int Quantity, decimal Total);
public record StockUnavailableEvent(int OrderId, string Sku);
public record PaymentProcessedEvent(int OrderId);
public record PaymentFailedEvent(int OrderId, string Sku, int Quantity, string Reason);

public sealed class InventoryModule(Dictionary<string, int> stock)
{
    public int Available(string sku) => stock.GetValueOrDefault(sku);

    public void Register(AsyncEventBus bus)
    {
        bus.Subscribe<OrderPlacedEvent>(async (e, ct) =>
        {
            if (Available(e.Sku) < e.Quantity)
            {
                await bus.PublishAsync(new StockUnavailableEvent(e.OrderId, e.Sku), ct);
                return;
            }
            stock[e.Sku] -= e.Quantity;
            Console.WriteLine($"[Inventario] Reservadas {e.Quantity} de {e.Sku} para la orden {e.OrderId}");
            await bus.PublishAsync(new StockReservedEvent(e.OrderId, e.Sku, e.Quantity, e.Total), ct);
        });

        // Compensación: si el pago falla, se devuelve el stock reservado.
        bus.Subscribe<PaymentFailedEvent>(e =>
        {
            stock[e.Sku] += e.Quantity;
            Console.WriteLine($"[Inventario] Liberadas {e.Quantity} de {e.Sku} (orden {e.OrderId}: {e.Reason})");
        });
    }
}

public sealed class PaymentsModule
{
    public void Register(AsyncEventBus bus) =>
        bus.Subscribe<StockReservedEvent>(async (e, ct) =>
        {
            if (e.Total > 5000m)
            {
                await bus.PublishAsync(new PaymentFailedEvent(e.OrderId, e.Sku, e.Quantity, "pago rechazado"), ct);
                return;
            }
            Console.WriteLine($"[Pagos] Cobrados {e.Total:N2} de la orden {e.OrderId}");
            await bus.PublishAsync(new PaymentProcessedEvent(e.OrderId), ct);
        });
}

// + AsyncEventBus del ejercicio 5
```

Salida (con cultura `en-US`):

```text
[Inventario] Reservadas 2 de keyboard para la orden 1
[Pagos] Cobrados 300.00 de la orden 1
[Pedidos] Orden 1 confirmada
---
[Inventario] Reservadas 1 de keyboard para la orden 2
[Inventario] Liberadas 1 de keyboard (orden 2: pago rechazado)
---
[Pedidos] Orden 3 rechazada: sin stock
Stock final de keyboard: 3
```

Esto es una **saga por coreografía**: ningún módulo coordina, cada uno reacciona y publica. La compensación (liberar stock) reemplaza a una transacción que abarque los tres módulos. Con microservicios reales, cada evento viajaría por un broker y aplicarían outbox e idempotencia ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md)).

</details>

### Reto 3: event bus "tipo producción"

**Misión:** mejora el `AsyncEventBus` con: nombre por suscripción, logging con `ILogger` (nivel de error con la excepción completa), medición del tiempo de cada handler y advertencia si un handler supera un umbral.

**Pista:** `Subscribe<T>(string name, Func<T, CancellationToken, Task> handler)`; `Stopwatch.GetTimestamp()`; el paquete `Microsoft.Extensions.Logging.Console` para tener un `ILogger` en consola.

<details>
<summary>Solución</summary>

Requiere `dotnet add package Microsoft.Extensions.Logging.Console`.

```csharp
using System.Diagnostics;
using Microsoft.Extensions.Logging;

using var loggerFactory = LoggerFactory.Create(b => b.AddSimpleConsole(o => o.SingleLine = true));
var bus = new ObservableEventBus(loggerFactory.CreateLogger<ObservableEventBus>(), slowThreshold: TimeSpan.FromMilliseconds(200));

bus.Subscribe<OrderCreatedEvent>("email", async (e, ct) => await Task.Delay(50, ct));
bus.Subscribe<OrderCreatedEvent>("reporte", async (e, ct) => await Task.Delay(350, ct));
bus.Subscribe<OrderCreatedEvent>("puntos", (e, ct) => throw new InvalidOperationException("servicio de puntos caído"));

await bus.PublishAsync(new OrderCreatedEvent(1, 100m));

public record OrderCreatedEvent(int OrderId, decimal Total);

public sealed class ObservableEventBus(ILogger<ObservableEventBus> logger, TimeSpan slowThreshold)
{
    private sealed record Registration(string Name, Func<object, CancellationToken, Task> Handler);

    private readonly Lock _lock = new();
    private readonly Dictionary<Type, Registration[]> _handlers = new();

    public IDisposable Subscribe<T>(string name, Func<T, CancellationToken, Task> handler) where T : notnull
    {
        var registration = new Registration(name, (e, ct) => handler((T)e, ct));
        lock (_lock)
            _handlers[typeof(T)] = [.. Get(typeof(T)), registration];

        return new Subscription(() =>
        {
            lock (_lock)
                _handlers[typeof(T)] = Get(typeof(T)).Where(r => !ReferenceEquals(r, registration)).ToArray();
        });
    }

    public async Task PublishAsync<T>(T @event, CancellationToken ct = default) where T : notnull
    {
        var type = @event.GetType();
        Registration[] registrations;
        lock (_lock)
            registrations = Get(type);

        logger.LogInformation("Publicando {Event} a {Count} handler(s)", type.Name, registrations.Length);
        await Task.WhenAll(registrations.Select(Run));

        async Task Run(Registration r)
        {
            var start = Stopwatch.GetTimestamp();
            try
            {
                await r.Handler(@event, ct);
                var elapsed = Stopwatch.GetElapsedTime(start);
                if (elapsed > slowThreshold)
                    logger.LogWarning("Handler {Handler} de {Event} lento: {Ms:N0} ms", r.Name, type.Name, elapsed.TotalMilliseconds);
                else
                    logger.LogDebug("Handler {Handler} de {Event} en {Ms:N0} ms", r.Name, type.Name, elapsed.TotalMilliseconds);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Handler {Handler} de {Event} falló", r.Name, type.Name);
            }
        }
    }

    private Registration[] Get(Type type) => _handlers.TryGetValue(type, out var r) ? r : [];

    private sealed class Subscription(Action unsubscribe) : IDisposable
    {
        private Action? _unsubscribe = unsubscribe;
        public void Dispose() => Interlocked.Exchange(ref _unsubscribe, null)?.Invoke();
    }
}
```

```text
info: ObservableEventBus[0] Publicando OrderCreatedEvent a 3 handler(s)
fail: ObservableEventBus[0] Handler puntos de OrderCreatedEvent falló System.InvalidOperationException: servicio de puntos caído ...
warn: ObservableEventBus[0] Handler reporte de OrderCreatedEvent lento: 351 ms
```

(El mensaje de `email` es de nivel `Debug` y no aparece con la configuración por defecto, que muestra desde `Information`.)

Lo que todavía le falta para ser "de producción" de verdad, y por qué en ese punto conviene un broker con una librería como MassTransit o Wolverine: persistencia (si el proceso se cae, los eventos en curso se pierden), reintentos con backoff, cola de mensajes fallidos, outbox e idempotencia.

</details>

-----

## Checkpoint

Antes de seguir, deberías poder responder sin mirar:

* ¿Por qué `.Cast<Func<T, Task>>()` sobre handlers registrados como `Action<T>` lanza `InvalidCastException`?
* ¿Qué le pasa a un evento publicado antes de que haya suscriptores en un bus en memoria?
* ¿Qué dos problemas resuelve la copia al escribir en el bus?
* ¿Por qué la factura debe reaccionar a `PaymentProcessedEvent` y no a `OrderCreatedEvent`?
* ¿Qué acoplamiento elimina un bus en memoria y cuál no?
* ¿Qué le falta a un bus en memoria para que ningún evento se pierda?
