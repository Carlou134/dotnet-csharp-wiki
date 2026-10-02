# Entrega confiable e idempotencia

## En una frase

Cuando los eventos viajan por un **broker** entre procesos, pueden **perderse**, **duplicarse** o **llegar desordenados**; los sistemas confiables lo asumen y se defienden con tres herramientas: el patrón **outbox** (no perder eventos), **consumidores idempotentes** (procesar los duplicados una sola vez) y **reintentos con cola de mensajes fallidos** (no trabarse con un mensaje roto).

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un evento y qué desacopla un broker: [Eventos y arquitectura dirigida por eventos](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md).
* Pub/Sub en memoria y sus límites: [Publish/Subscribe en memoria](02-Publish%20Subscribe%20en%20memoria.md).
* Agregados y transacciones: [Agregados](../01-domain-driven-design/04-Agregados.md).
* `await foreach` e `IAsyncEnumerable`: [Concurrencia, cancelación y errores](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/02-Concurrencia%20cancelacion%20y%20errores.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Broker de mensajes:** servidor que recibe, guarda y entrega mensajes (RabbitMQ, Azure Service Bus, Amazon SQS/SNS, Kafka).
* **Cola (*queue*) y tópico (*topic*):** una cola entrega cada mensaje a **un** consumidor; un tópico lo entrega a **cada** suscripción.
* **Acuse de recibo (*ack*):** el consumidor avisa al broker que terminó con un mensaje; si no lo hace, el broker lo vuelve a entregar.
* **Al menos una vez (*at-least-once*):** garantía de entrega en la que ningún mensaje se pierde, pero alguno puede llegar repetido.
* **Idempotente:** procesar el mismo mensaje una o varias veces deja el mismo resultado.
* **Outbox (bandeja de salida):** tabla donde se guardan los eventos en la misma transacción que el cambio de datos, para publicarlos después.
* **Cola de mensajes fallidos (*dead-letter queue*, DLQ):** lugar donde terminan los mensajes que no se pudieron procesar, para revisarlos aparte.

-----

## El problema

El servicio de pedidos guarda el pedido y publica el evento:

```csharp
await db.Pedidos.AddAsync(pedido);
await db.SaveChangesAsync();                        // 1. se guarda el pedido
await broker.PublishAsync(new PedidoCreado(...));   // 2. se publica el evento
```

Y el servicio de pagos cobra al recibirlo:

```csharp
async Task Manejar(PedidoCreado e) => await pasarela.CobrarAsync(e.PedidoId, e.Total);
```

Lo que puede salir mal:

| Momento | Qué pasa | Consecuencia |
| --- | --- | --- |
| El proceso se cae **entre 1 y 2** | El pedido existe, el evento nunca se publica | Pedido que nadie cobra ni despacha |
| Se invierte el orden (publicar y luego guardar) y falla el guardado | El evento existe, el pedido no | Se cobra un pedido inexistente |
| Pagos cobra, pero se cae antes del *ack* | El broker reentrega el mensaje | **Se cobra dos veces** |
| La pasarela da *timeout* una vez | Si no hay reintento, el pedido queda sin cobrar | Si se reintenta sin control, se cobra dos veces |
| Un evento tiene datos inválidos | El consumidor falla, el broker reentrega, falla, reentrega... | Un mensaje "venenoso" bloquea la cola |

Escribir en la base de datos y en el broker son **dos sistemas distintos**: no existe una transacción que cubra a los dos (es el problema de la **doble escritura**).

-----

## Cómo funciona

### 1. Garantías de entrega

| Garantía | Cómo se logra | Riesgo |
| --- | --- | --- |
| **Como máximo una vez** | *Ack* al recibir, antes de procesar | Si el consumidor falla, el mensaje **se pierde** |
| **Al menos una vez** | *Ack* después de procesar con éxito | Si falla antes del *ack*, **se repite** |
| **Exactamente una vez** | No existe de punta a punta en sistemas distribuidos | Se **simula**: al menos una vez + consumidor idempotente |

La regla práctica: **diseña para "al menos una vez" y haz que los consumidores sean idempotentes**. Así ningún mensaje se pierde y los duplicados no hacen daño.

### 2. Outbox: no perder eventos

```text
 Servicio de pedidos                                    Broker
 ┌─────────────────────────────────────────┐
 │  UNA transacción de base de datos:      │
 │    INSERT INTO Pedidos (...)            │
 │    INSERT INTO Outbox  (evento, enviado=0)
 └─────────────────────────────────────────┘
                    │
                    ▼  proceso en segundo plano (relay)
 ┌─────────────────────────────────────────┐   publica    ┌──────────┐
 │  SELECT * FROM Outbox WHERE enviado = 0 │ ───────────► │  tópico  │ ──► consumidores
 │  UPDATE Outbox SET enviado = 1          │              └──────────┘
 └─────────────────────────────────────────┘
```

* El pedido y el evento se guardan **juntos o ninguno**: no hay pedido sin evento ni evento sin pedido.
* Un proceso aparte (*relay*) publica los eventos pendientes y los marca como enviados.
* Si el relay se cae **después de publicar y antes de marcar**, al reiniciar vuelve a publicar ese evento: la outbox garantiza **al menos una vez**, así que el consumidor debe ser idempotente.

### 3. Consumidor idempotente

```csharp
if (await db.EventosProcesados.AnyAsync(e => e.Id == evento.EventoId)) return;   // ya lo procesé

await using var tx = await db.Database.BeginTransactionAsync();
db.Cobros.Add(new Cobro(evento.PedidoId, evento.Total));
db.EventosProcesados.Add(new EventoProcesado(evento.EventoId));   // en la MISMA transacción
await db.SaveChangesAsync();
await tx.CommitAsync();
```

* Por eso el evento lleva un `EventoId`: identifica la ocurrencia, no el pedido.
* Registrar el Id **en la misma transacción** que el efecto es lo que lo hace confiable. Si se registra después y el proceso se cae entre medio, el siguiente intento repite el efecto.
* Una clave única en la tabla de eventos procesados protege incluso si dos instancias reciben el mismo mensaje a la vez.
* Algunas operaciones son idempotentes por naturaleza ("marcar el pedido como pagado" frente a "sumar 1 al contador"): cuando puedas, diseña así los efectos.

### 4. Reintentos y cola de mensajes fallidos

No todos los errores son iguales:

| Tipo de error | Ejemplo | Qué hacer |
| --- | --- | --- |
| **Transitorio** | *Timeout*, servicio no disponible, bloqueo de base de datos | Reintentar con espera creciente (*backoff*) y un máximo |
| **Permanente** | Datos inválidos, versión de evento desconocida, un bug | No reintentar: enviarlo a la **cola de mensajes fallidos** y alertar |

Sin DLQ, un mensaje imposible de procesar se reintenta para siempre y bloquea a los que vienen detrás. Con DLQ, se aparta, el resto sigue y alguien lo revisa (y lo reenvía después de corregir el problema).

### 5. Orden

Los brokers no garantizan el orden entre mensajes en general (Kafka lo garantiza dentro de una partición; Service Bus, dentro de una sesión). Diseña los consumidores para tolerarlo: por ejemplo, ignorar un `PedidoActualizado` con una versión menor que la ya procesada.

### 6. Herramientas en .NET

* **Brokers:** RabbitMQ, Azure Service Bus, Amazon SQS/SNS, Apache Kafka.
* **Librerías:** MassTransit, Wolverine, NServiceBus y Rebus implementan outbox, reintentos, DLQ, sagas y serialización sobre esos brokers. Escribirlo todo a mano es un buen ejercicio, pero en producción conviene una librería probada. Revisa las licencias: varias librerías populares del ecosistema .NET (como MediatR y MassTransit 9) pasaron a modelos comerciales en 2025.
* **En memoria:** `System.Threading.Channels` sirve como cola productor/consumidor dentro de un proceso (por ejemplo, con un `BackgroundService`), pero no persiste nada.

-----

## Ejemplo completo

Una simulación en consola: la "base de datos" tiene una outbox, el broker es un `Channel<T>`, el relay se cae una vez y el consumidor de pagos reintenta, descarta duplicados y aparta los mensajes inválidos.

```csharp
using System.Threading.Channels;

var db = new BaseDeDatos();
db.CrearPedido(1, 100m);
db.CrearPedido(2, 250m);
db.CrearPedido(3, -5m);     // dato corrupto
db.CrearPedido(4, 40m);

var broker = Channel.CreateUnbounded<PedidoCreado>();

Console.WriteLine("== Relay: primera ejecución ==");
await Relay.PublicarPendientesAsync(db, broker.Writer, simularCaida: true);
Console.WriteLine("== Relay: se reinicia ==");
await Relay.PublicarPendientesAsync(db, broker.Writer, simularCaida: false);
broker.Writer.Complete();

Console.WriteLine("== Consumidor de pagos ==");
var pagos = new ConsumidorDePagos();
await foreach (var evento in broker.Reader.ReadAllAsync())
    await pagos.ProcesarAsync(evento);

Console.WriteLine($"Cobros realizados: {string.Join(", ", pagos.Cobros)}");
Console.WriteLine($"Cola de mensajes fallidos: pedido(s) {string.Join(", ", pagos.Fallidos.Select(e => e.PedidoId))}");

public record PedidoCreado(Guid EventoId, int PedidoId, decimal Total);

public sealed class BaseDeDatos
{
    private readonly List<(PedidoCreado Evento, bool Enviado)> _outbox = [];

    public void CrearPedido(int id, decimal total)
    {
        // En una base real: INSERT INTO Pedidos + INSERT INTO Outbox, en la MISMA transacción.
        _outbox.Add((new PedidoCreado(Guid.NewGuid(), id, total), false));
        Console.WriteLine($"[Pedidos] Pedido {id} y su evento guardados juntos");
    }

    public IReadOnlyList<PedidoCreado> Pendientes() =>
        _outbox.Where(o => !o.Enviado).Select(o => o.Evento).ToList();

    public void MarcarEnviado(Guid eventoId)
    {
        var i = _outbox.FindIndex(o => o.Evento.EventoId == eventoId);
        _outbox[i] = (_outbox[i].Evento, Enviado: true);
    }
}

public static class Relay
{
    public static async Task PublicarPendientesAsync(BaseDeDatos db, ChannelWriter<PedidoCreado> broker, bool simularCaida)
    {
        var pendientes = db.Pendientes();
        for (var i = 0; i < pendientes.Count; i++)
        {
            await broker.WriteAsync(pendientes[i]);
            Console.WriteLine($"[Relay] Publicado el evento del pedido {pendientes[i].PedidoId}");

            if (simularCaida && i == pendientes.Count - 1)
            {
                Console.WriteLine("[Relay] Se cayó antes de marcarlo como enviado");
                return;
            }

            db.MarcarEnviado(pendientes[i].EventoId);
        }
    }
}

public sealed class ConsumidorDePagos
{
    private const int MaximoIntentos = 3;
    private readonly HashSet<Guid> _procesados = [];
    private bool _pasarelaYaFallo;

    public List<string> Cobros { get; } = [];
    public List<PedidoCreado> Fallidos { get; } = [];

    public async Task ProcesarAsync(PedidoCreado e)
    {
        if (_procesados.Contains(e.EventoId))
        {
            Console.WriteLine($"[Pagos] Evento del pedido {e.PedidoId} duplicado: se ignora");
            return;
        }

        for (var intento = 1; intento <= MaximoIntentos; intento++)
        {
            try
            {
                Cobrar(e);
                _procesados.Add(e.EventoId);   // en una base real: en la misma transacción que el cobro
                return;
            }
            catch (TimeoutException) when (intento < MaximoIntentos)
            {
                Console.WriteLine($"[Pagos] Pedido {e.PedidoId}: timeout en el intento {intento}, reintentando...");
                await Task.Delay(100 * intento);   // backoff creciente
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[Pagos] Pedido {e.PedidoId}: {ex.Message} → cola de mensajes fallidos");
                Fallidos.Add(e);
                return;
            }
        }
    }

    private void Cobrar(PedidoCreado e)
    {
        if (e.Total <= 0) throw new ArgumentException($"total inválido ({e.Total})");

        if (e.PedidoId == 2 && !_pasarelaYaFallo)   // simula un fallo transitorio de la pasarela
        {
            _pasarelaYaFallo = true;
            throw new TimeoutException();
        }

        Cobros.Add($"#{e.PedidoId} ({e.Total:N2})");
        Console.WriteLine($"[Pagos] Cobrado el pedido {e.PedidoId}: {e.Total:N2}");
    }
}
```

Salida (con cultura `en-US`):

```text
[Pedidos] Pedido 1 y su evento guardados juntos
[Pedidos] Pedido 2 y su evento guardados juntos
[Pedidos] Pedido 3 y su evento guardados juntos
[Pedidos] Pedido 4 y su evento guardados juntos
== Relay: primera ejecución ==
[Relay] Publicado el evento del pedido 1
[Relay] Publicado el evento del pedido 2
[Relay] Publicado el evento del pedido 3
[Relay] Publicado el evento del pedido 4
[Relay] Se cayó antes de marcarlo como enviado
== Relay: se reinicia ==
[Relay] Publicado el evento del pedido 4
== Consumidor de pagos ==
[Pagos] Cobrado el pedido 1: 100.00
[Pagos] Pedido 2: timeout en el intento 1, reintentando...
[Pagos] Cobrado el pedido 2: 250.00
[Pagos] Pedido 3: total inválido (-5) → cola de mensajes fallidos
[Pagos] Cobrado el pedido 4: 40.00
[Pagos] Evento del pedido 4 duplicado: se ignora
Cobros realizados: #1 (100.00), #2 (250.00), #4 (40.00)
Cola de mensajes fallidos: pedido(s) 3
```

Las cuatro defensas en acción:

* **Outbox:** ningún pedido quedó sin evento, aunque el relay se cayó.
* **Al menos una vez:** el evento del pedido 4 se publicó dos veces.
* **Idempotencia:** el pedido 4 se cobró una sola vez.
* **Reintento + DLQ:** el timeout del pedido 2 se recuperó; el pedido 3, con datos inválidos, se apartó sin bloquear al 4.

-----

## Errores comunes

**1. Guardar y publicar por separado.**
Qué pasa: pedidos sin evento (si el proceso se cae después de guardar) o eventos sin pedido (si se publica primero).
Por qué: la base de datos y el broker no comparten transacción.
Arreglo: patrón outbox.

**2. Consumidores que no son idempotentes.**
Qué pasa: cobros, emails o descuentos de stock duplicados.
Por qué: se asume que cada mensaje llega una vez.
Arreglo: registrar el `EventoId` procesado en la misma transacción que el efecto, con clave única.

**3. Marcar como procesado fuera de la transacción.**
Qué pasa: si el proceso se cae entre el efecto y la marca, el reintento repite el efecto.
Por qué: dos escrituras separadas.
Arreglo: efecto + marca en una sola transacción.

**4. Reintentar todo, siempre.**
Qué pasa: un mensaje con datos inválidos se reintenta sin fin y bloquea la cola.
Por qué: no se distingue error transitorio de permanente.
Arreglo: reintentos con máximo y backoff solo para errores transitorios; los permanentes, a la DLQ.

**5. Asumir orden.**
Qué pasa: llega `PedidoCancelado` antes que `PedidoCreado` y el consumidor falla o deja datos incoherentes.
Por qué: el broker no garantiza orden entre mensajes.
Arreglo: consumidores tolerantes (versiones, estados), o particiones/sesiones por clave cuando el orden importa.

**6. Usar un `Channel<T>` como si fuera un broker.**
Qué pasa: al reiniciar la aplicación se pierden todos los mensajes en la cola.
Por qué: un canal vive en memoria.
Arreglo: `Channel<T>` para trabajo en segundo plano dentro del proceso; un broker persistente para eventos que no pueden perderse.

-----

## Según la versión de .NET

* **.NET Core 3.0:** `System.Threading.Channels` incluido; `await foreach` e `IAsyncEnumerable` (C# 8).
* **.NET Core 3.0+:** `BackgroundService` / `IHostedService` para procesos en segundo plano como el relay de una outbox.
* **.NET 8:** `Microsoft.Extensions.Resilience` (sobre Polly 8) para reintentos con backoff, *circuit breakers* y *timeouts*.
* **C# 12:** expresiones de colección (`[]`) usadas en el ejemplo.

-----

## Cuándo sí y cuándo no

**Necesitas outbox + idempotencia cuando:**

* Los eventos cruzan procesos o servicios a través de un broker.
* Perder o duplicar un evento tiene consecuencias de negocio (dinero, stock, comunicaciones al cliente).

**Puedes simplificar cuando:**

* Los eventos son internos de un proceso y se manejan en la misma transacción (eventos de dominio).
* El efecto es naturalmente idempotente y perder un evento ocasional es aceptable (por ejemplo, refrescar una caché).

-----

## Resumen en 5 líneas

1. Entre procesos, los mensajes pueden perderse, duplicarse o desordenarse: diseña para eso.
2. Garantía realista: al menos una vez + consumidores idempotentes.
3. Outbox: el cambio de datos y el evento se guardan en la misma transacción; un relay publica después.
4. Idempotencia: registra el `EventoId` procesado en la misma transacción que el efecto.
5. Reintenta solo los errores transitorios, con backoff y máximo; los permanentes van a la DLQ.

-----

## Para profundizar

<details>
<summary>Inbox: la outbox del lado del consumidor</summary>

El patrón **inbox** guarda primero el mensaje recibido en una tabla (con el `EventoId` como clave única) y lo procesa después desde ahí. Combina la deduplicación con la posibilidad de reprocesar y auditar lo recibido. Librerías como MassTransit y Wolverine implementan outbox e inbox sobre EF Core.

</details>

<details>
<summary>Sagas y compensaciones</summary>

Si un proceso cruza varios servicios (reservar stock → cobrar → despachar) y un paso falla, no hay una transacción que deshaga los anteriores. Una **saga** define, para cada paso, una **acción compensatoria** (si el cobro falla, liberar el stock) y coordina el flujo con eventos o comandos. Es la forma de lograr consistencia sin transacciones distribuidas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Cuando se usan mensajes entre servicios, un mensaje puede llegar dos veces o perderse si algo se cae. Para no perderlos se usa el patrón outbox: el evento se guarda en la base de datos junto con el cambio, y otro proceso lo publica después. Para que los duplicados no hagan daño, el consumidor guarda los Ids de los eventos que ya procesó y los ignora si llegan de nuevo.

### Respuesta ampliada (semi-senior)

Asumo entrega al menos una vez: el *ack* va después del procesamiento, así que puede haber duplicados. Del lado del productor resuelvo la doble escritura con un outbox transaccional y un relay en segundo plano; del lado del consumidor, idempotencia registrando el `EventoId` en la misma transacción que el efecto, con clave única, o con efectos idempotentes por diseño. Distingo errores transitorios (reintentos con backoff exponencial y jitter, con un máximo) de permanentes (directo a la DLQ con alertas). No asumo orden salvo con particiones o sesiones por clave. En .NET uso MassTransit, Wolverine o NServiceBus sobre RabbitMQ o Service Bus, que implementan outbox, inbox, reintentos y sagas, revisando sus licencias.

### Preguntas frecuentes de seguimiento

**1. ¿Existe la entrega exactamente una vez?**
No de punta a punta en un sistema distribuido; se logra el efecto con entrega al menos una vez más consumidores idempotentes.

**2. ¿Qué problema resuelve el outbox?**
La doble escritura: guardar datos y publicar el evento sin una transacción común. Con el outbox, ambos quedan en la misma transacción de la base de datos.

**3. ¿Qué es una DLQ?**
Una cola donde van los mensajes que no se pudieron procesar tras los reintentos o por errores permanentes, para revisarlos sin bloquear al resto.

-----

## Práctica

**Ejercicio 1.** Para cada consumidor, indica si es idempotente por naturaleza y, si no, cómo lo harías idempotente:

1. "Marcar el pedido como pagado".
2. "Sumar 10 puntos al cliente".
3. "Enviar un email de confirmación".
4. "Fijar el stock del producto en 25".

<details>
<summary>Solución</summary>

1. **Idempotente:** marcarlo dos veces deja el mismo estado.
2. **No:** dos ejecuciones suman 20. Registrar el `EventoId` procesado en la misma transacción que la suma (o guardar un movimiento de puntos con el `EventoId` como clave única).
3. **No:** el cliente recibe dos emails. Registrar el envío por `EventoId` antes de enviarlo (aceptando que, si el proceso se cae entre el registro y el envío, ese email se pierde) o usar un proveedor que acepte una clave de idempotencia.
4. **Idempotente** si el evento trae el valor final; pero cuidado con el orden: un evento viejo que llega tarde podría pisar uno nuevo. Incluir una versión y descartar las anteriores.

</details>

**Ejercicio 2.** El relay de la outbox publica cada evento y luego lo marca como enviado. ¿Qué pasa si se invierte el orden (marca primero y publica después)?

<details>
<summary>Solución</summary>

Si el proceso se cae entre marcar y publicar, el evento queda como enviado pero **nunca se publicó**: se pierde. El orden "publicar, luego marcar" puede duplicar (si se cae entre ambos), pero nunca pierde, y los duplicados ya los maneja el consumidor idempotente. Entre perder y duplicar, se elige duplicar.

</details>

-----

## Siguiente lección

Terminaste el módulo. Para practicar con los ejercicios de clase: [Ejercicios de eventos y Pub/Sub](Ejercicios.md). Vuelve al [índice del módulo](README.md).
