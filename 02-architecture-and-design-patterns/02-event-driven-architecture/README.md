# Arquitectura dirigida por eventos (EDA) y Publish/Subscribe

En esta carpeta aprendes a comunicar partes de un sistema con **eventos** en lugar de llamadas directas: qué es un evento y qué desacopla (y qué no), cómo construir un bus **Publish/Subscribe** en memoria que no se rompa con errores, hilos o handlers asíncronos, y cómo se logra una **entrega confiable** entre procesos con outbox, consumidores idempotentes, reintentos y colas de mensajes fallidos.

-----

## Antes de empezar

Conviene que ya tengas:

* Delegados y eventos de C#: [Delegados y eventos](../../01-csharp-core-and-runtime/09-delegados-y-eventos/README.md).
* `async`/`await`, `Task.WhenAll`, cancelación: [Asincronía y archivos](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).
* Agregados y eventos de dominio: [DDD táctico](../01-domain-driven-design/README.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Eventos y arquitectura dirigida por eventos](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md) | Evento frente a comando, tipos de acoplamiento, costos de EDA, eventos de dominio y de integración | Delegados y eventos, DDD |
| [2. Publish/Subscribe en memoria](02-Publish%20Subscribe%20en%20memoria.md) | Bus tipado, aislamiento de errores, copia al escribir, `IDisposable`, handlers asíncronos, secuencial o concurrente | Eventos, asincronía |
| [3. Entrega confiable e idempotencia](03-Entrega%20confiable%20e%20idempotencia.md) | Brokers, garantías de entrega, outbox, consumidores idempotentes, reintentos y DLQ, orden | Pub/Sub en memoria, agregados |

Para practicar más: [Ejercicios de eventos y Pub/Sub](Ejercicios.md), con los 5 ejercicios y 3 retos de la Sesión 11 corregidos (todos con la solución plegada).

-----

## El mapa completo en una mirada

```
Evento              hecho inmutable, en pasado (PedidoCreado), 0..N consumidores, no se rechaza
Comando             pedido en imperativo (CobrarPedido), 1 destinatario, puede rechazarse
Desacopla           conocimiento (siempre) · tiempo (solo con broker)
No desacopla        el CONTRATO del evento: trátalo como una API pública
Cuesta              consistencia eventual · duplicados · desorden · flujos más difíciles de seguir

Bus en memoria      Dictionary<Type, handler[]> · copia al escribir bajo lock · GetType()
                    Subscribe → IDisposable · try/catch por handler · async con Func<T, CancellationToken, Task>
Límite              se cae el proceso → se pierden los eventos

Entre procesos      al menos una vez + consumidor idempotente (EventoId en la misma transacción)
Outbox              datos + evento en UNA transacción · relay publica y luego marca
Errores             transitorios → reintento con backoff · permanentes → cola de mensajes fallidos
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura que en [C# Core](../../01-csharp-core-and-runtime/README.md): en una frase, antes de empezar, el problema, cómo funciona, ejemplo completo, errores comunes, según la versión, cuándo sí y cuándo no, resumen en 5 líneas, para profundizar, en entrevista, práctica y siguiente lección.

Los ejemplos completos son aplicaciones de consola (.NET 10).

-----

## Después de esta carpeta

Continúa con [Resiliencia de servicios con Polly](../03-resiliencia-de-servicios/README.md) para proteger llamadas remotas con retry, circuit breaker y timeouts. Después puedes llevar los eventos a un broker real (RabbitMQ o Azure Service Bus con MassTransit o Wolverine), procesar en segundo plano con `BackgroundService` y coordinar procesos largos con sagas.
