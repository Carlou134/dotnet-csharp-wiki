# Eventos y arquitectura dirigida por eventos

## En una frase

En una **arquitectura dirigida por eventos** (EDA, *Event-Driven Architecture*), un componente anuncia **lo que ya pasó** ("se creó el pedido 42") y otros reaccionan por su cuenta, sin que el que anuncia sepa quiénes son ni cuántos.

-----

## Antes de empezar

Conviene que ya sepas:

* Delegados y eventos de C# (`event`, suscribirse con `+=`): [Eventos](../../01-csharp-core-and-runtime/09-delegados-y-eventos/02-Eventos.md).
* Agregados y eventos de dominio: [Agregados](../01-domain-driven-design/04-Agregados.md).
* Principio abierto/cerrado (SOLID): [Principios DRY KISS y refactoring](../../01-csharp-core-and-runtime/11-codigo-limpio/02-Principios%20DRY%20KISS%20y%20refactoring.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Evento:** mensaje inmutable que describe un hecho que **ya ocurrió**. Se nombra en pasado: `PedidoCreado`.
* **Comando:** mensaje que **pide** que algo ocurra. Se nombra en imperativo: `CrearPedido`. Puede ser rechazado.
* **Productor (*publisher*):** el componente que publica un evento.
* **Consumidor (*subscriber*, *handler*):** el componente que reacciona a un evento.
* **Broker / bus de eventos:** intermediario que recibe los eventos y los entrega a los consumidores.
* **Acoplamiento:** cuánto necesita saber un componente de otro para funcionar.
* **Consistencia eventual:** los datos de distintas partes del sistema terminan siendo coherentes, pero no en el mismo instante.

-----

## El problema

Al crear un pedido hay que enviar un email, reservar stock y registrar analítica:

```csharp
public class ServicioDePedidos(IEmailService email, IInventario inventario, IAnalitica analitica)
{
    public void Crear(Pedido pedido)
    {
        _repositorio.Guardar(pedido);
        email.EnviarConfirmacion(pedido);       // si el servidor SMTP tarda 10 s, el usuario espera 10 s
        inventario.Reservar(pedido);            // si falla, ¿se deshace el pedido? ¿y el email ya enviado?
        analitica.Registrar(pedido);
    }
}
```

Problemas:

* **El servicio de pedidos conoce a todos.** Cada reacción nueva (programa de puntos, alerta de fraude, factura) obliga a **modificarlo**, agregar una dependencia y volver a probarlo.
* **Acoplamiento temporal:** el pedido no termina hasta que terminan todos. Si el email está caído, crear pedidos falla.
* **Una responsabilidad que no es suya:** "crear un pedido" terminó mezclado con email, stock y analítica.

-----

## Cómo funciona

### 1. Llamada directa frente a evento

```text
 LLAMADA DIRECTA                                   EVENTO
 ───────────────                                   ──────
 ServicioDePedidos                                 ServicioDePedidos
   ├──► Email.Enviar(...)                            └──► publica PedidoCreado ──► [ bus / broker ]
   ├──► Inventario.Reservar(...)                                                      │
   └──► Analitica.Registrar(...)                         ┌────────────────────────────┼──────────────┐
                                                         ▼                            ▼              ▼
 Pedidos conoce a los 3, los llama en orden          Email               Inventario        Analítica
 y espera a cada uno.                              (cada uno se suscribe por su cuenta; Pedidos no los conoce)
```

Con eventos, `ServicioDePedidos` solo dice **"pasó esto"**. Agregar una reacción nueva es agregar un suscriptor: el productor **no cambia** (principio abierto/cerrado a nivel de arquitectura).

### 2. Evento, comando y consulta

| | Evento | Comando | Consulta |
| --- | --- | --- | --- |
| Significa | "Pasó X" | "Haz X" | "Dime X" |
| Nombre | Pasado: `PedidoCreado` | Imperativo: `CobrarPedido` | `ObtenerPedido` |
| Destinatarios | 0, 1 o muchos | Exactamente 1 | Exactamente 1 |
| ¿Se puede rechazar? | No: ya ocurrió | Sí | — |
| El emisor espera respuesta | No | A veces | Sí |

Confundirlos produce diseños raros: publicar un "evento" `ProcesarPago` es en realidad un comando disfrazado. Si **exactamente un** componente **debe** hacer algo y el emisor necesita saber si salió bien, es un comando.

### 3. Cómo se escribe un evento en C#

```csharp
public record PedidoCreado(
    Guid EventoId,          // identifica ESTE evento: permite detectar duplicados
    DateTimeOffset OcurrioEn,
    int PedidoId,
    int ClienteId,
    decimal Total);
```

* **`record`:** inmutable y con igualdad por valor; un hecho no cambia.
* **En pasado y con lenguaje del negocio:** `PedidoCreado`, no `PedidoInsertadoEnTabla`.
* **Solo los datos necesarios:** los Ids y lo que los consumidores suelen necesitar. No el agregado entero.
* **Un `EventoId`:** cuando hay un broker real, el mismo evento puede llegar dos veces; con el Id, el consumidor lo detecta ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md)).

### 4. Qué desacopla y qué no

Es tentador decir que los eventos dan "desacoplamiento total". No es así: **cambian** el tipo de acoplamiento.

| Tipo de acoplamiento | Llamada directa | Eventos en memoria (mismo proceso) | Eventos con broker (RabbitMQ, Service Bus, Kafka) |
| --- | --- | --- | --- |
| **Conocimiento** (quién reacciona) | Alto | **Ninguno** | **Ninguno** |
| **Temporal** (todos deben estar disponibles a la vez) | Alto | Alto: todo corre en el mismo proceso | **Bajo**: el broker guarda el mensaje |
| **Contrato** (la forma de los datos) | La firma del método | **La forma del evento** | **La forma del evento**, y ahora versionada entre servicios |

El acoplamiento al contrato **sigue existiendo**: si renombras `Total` a `Importe`, rompes a todos los consumidores (que ni siquiera conoces). Los eventos son un contrato público, como una API.

### 5. El precio de los eventos

| Ganas | Pagas |
| --- | --- |
| Agregar reacciones sin tocar el productor | El flujo ya no se lee de arriba abajo: "¿quién reacciona a esto?" se responde buscando suscriptores |
| Con broker: el productor no espera ni depende de que los consumidores estén vivos | **Consistencia eventual**: el stock se reserva "un poco después" que el pedido |
| Escalar consumidores por separado | Mensajes duplicados, desordenados o perdidos que hay que manejar |
| | Depurar y rastrear un flujo que cruza varios procesos (necesitas *correlation ids* y trazas) |

### 6. Eventos de dominio y eventos de integración

* **Evento de dominio:** ocurre dentro de un servicio (un agregado lo registra) y se maneja en el mismo proceso, a menudo en la misma transacción. Ejemplo: `PedidoConfirmado` dispara el cálculo de puntos en el mismo servicio.
* **Evento de integración:** cruza los límites del servicio, viaja por un broker y lo consumen otros sistemas. Es un **contrato público** y se versiona con cuidado.

Un evento de dominio puede convertirse en uno de integración, pero no deberían ser la misma clase: el de integración solo lleva lo que otros necesitan saber.

-----

## Ejemplo completo

Antes y después: el servicio de pedidos publica un evento, y agregar una reacción no lo modifica.

```csharp
var suscriptores = new List<Action<PedidoCreado>>();
var pedidos = new ServicioDePedidos(publicar: e => suscriptores.ForEach(s => s(e)));

suscriptores.Add(e => Console.WriteLine($"[Email]      Confirmación del pedido {e.PedidoId} al cliente {e.ClienteId}"));
suscriptores.Add(e => Console.WriteLine($"[Inventario] Reservando stock del pedido {e.PedidoId}"));

pedidos.Crear(pedidoId: 1, clienteId: 7, total: 250m);

Console.WriteLine("--- se agrega analítica: ServicioDePedidos no cambia ---");
suscriptores.Add(e => Console.WriteLine($"[Analítica]  Venta de {e.Total:N2} registrada"));

pedidos.Crear(pedidoId: 2, clienteId: 9, total: 80m);

public record PedidoCreado(Guid EventoId, DateTimeOffset OcurrioEn, int PedidoId, int ClienteId, decimal Total);

public class ServicioDePedidos(Action<PedidoCreado> publicar)
{
    public void Crear(int pedidoId, int clienteId, decimal total)
    {
        // ... validar y guardar el pedido ...
        Console.WriteLine($"[Pedidos]    Pedido {pedidoId} guardado");
        publicar(new PedidoCreado(Guid.NewGuid(), DateTimeOffset.UtcNow, pedidoId, clienteId, total));
    }
}
```

Salida (con cultura `en-US`):

```text
[Pedidos]    Pedido 1 guardado
[Email]      Confirmación del pedido 1 al cliente 7
[Inventario] Reservando stock del pedido 1
--- se agrega analítica: ServicioDePedidos no cambia ---
[Pedidos]    Pedido 2 guardado
[Email]      Confirmación del pedido 2 al cliente 9
[Inventario] Reservando stock del pedido 2
[Analítica]  Venta de 80.00 registrada
```

Fíjate en lo que **no** se resolvió todavía: todo corre en el mismo hilo y en orden. Si el email tarda o falla, el pedido también. Es desacoplamiento de **conocimiento**, no temporal. Las lecciones siguientes construyen un bus más robusto y luego muestran la entrega con un broker.

-----

## Errores comunes

**1. Eventos con nombre de comando.**
Qué pasa: `ProcesarPago` publicado como evento; nadie sabe si alguien lo procesó ni qué pasa si nadie lo hace.
Por qué: se usa el bus para "llamar" a otro componente sin dependencia directa.
Arreglo: los eventos en pasado (`PedidoCreado`); si un componente concreto **debe** hacer algo y el emisor necesita el resultado, es un comando.

**2. Eventos con el agregado entero.**
Qué pasa: `PedidoCreado(Pedido pedido)` con todas sus propiedades y colecciones.
Por qué: es lo más cómodo.
Arreglo: solo los datos necesarios (Ids y valores clave). Cada campo publicado es parte del contrato y no se puede quitar sin romper a alguien.

**3. Eventos mutables.**
Qué pasa: un consumidor modifica el evento y el siguiente recibe datos alterados.
Por qué: clase con `set` públicos.
Arreglo: `record` con propiedades de solo lectura.

**4. Creer que los eventos eliminan todo el acoplamiento.**
Qué pasa: se cambia la forma del evento "porque nadie depende de mí" y se rompen consumidores desconocidos.
Por qué: el acoplamiento se movió al contrato.
Arreglo: trata los eventos como una API pública: agrega campos, no los quites ni los renombres, versiona si hace falta.

**5. Usar eventos para todo.**
Qué pasa: un flujo de 3 pasos que antes se leía en 10 líneas ahora está repartido en 6 archivos.
Por qué: EDA por moda.
Arreglo: si el flujo es simple, local y necesita una respuesta inmediata, una llamada directa es más clara.

-----

## Según la versión de C#

* **C# 1:** delegados y la palabra clave `event`, la versión más básica de publicar/suscribir dentro de un objeto.
* **C# 9:** `record`, ideal para eventos inmutables.
* **C# 12:** constructores primarios (`ServicioDePedidos(Action<PedidoCreado> publicar)`).
* **.NET Core 3.0:** `System.Threading.Channels`, base para colas en memoria con productor y consumidor.

-----

## Cuándo sí y cuándo no

**Usa eventos cuando:**

* Varias partes del sistema reaccionan al mismo hecho y quieres agregar reacciones sin tocar el origen.
* La reacción puede ocurrir después (enviar un email, actualizar un reporte, sincronizar otro sistema).
* Integras servicios distintos y no quieres que uno caiga porque otro está caído (con un broker).

**Usa una llamada directa cuando:**

* Necesitas la respuesta para continuar (validar stock antes de confirmar una compra).
* El flujo es simple y está todo en el mismo módulo.
* Necesitas consistencia inmediata y una sola transacción.

-----

## Resumen en 5 líneas

1. Un evento es un hecho inmutable en pasado; un comando es un pedido en imperativo.
2. El productor publica sin conocer a los consumidores; agregar reacciones no lo modifica.
3. Los eventos eliminan el acoplamiento de conocimiento, pero el contrato del evento sigue acoplando.
4. Solo un broker elimina el acoplamiento temporal, a cambio de consistencia eventual, duplicados y desorden.
5. Úsalos donde varias reacciones pueden ocurrir después; para respuestas inmediatas, llamadas directas.

-----

## Para profundizar

<details>
<summary>Coreografía y orquestación</summary>

Cuando un proceso de negocio cruza varios servicios (pedido → stock → pago → envío):

* **Coreografía:** cada servicio reacciona a eventos y publica los suyos (`StockReservado` → Pagos cobra → `PagoRealizado` → Envíos despacha). No hay un coordinador; el flujo "emerge". Fácil de extender, difícil de seguir.
* **Orquestación:** un coordinador (una *saga* o *process manager*) envía comandos a cada servicio y decide el siguiente paso según las respuestas. El flujo está en un solo lugar; el coordinador es un punto central.

Muchos sistemas combinan ambas: coreografía para reacciones simples y orquestación para procesos con compensaciones (si el pago falla, liberar el stock).

</details>

<details>
<summary>Event sourcing no es lo mismo que EDA</summary>

**Event sourcing** guarda el estado de una entidad como la secuencia de eventos que le ocurrieron (en lugar de guardar solo el estado actual). Se puede hacer EDA sin event sourcing (lo más común) y event sourcing sin publicar eventos a otros servicios. Son ideas relacionadas, pero independientes.

</details>

-----

## En entrevista

### Respuesta corta (junior)

En una arquitectura dirigida por eventos, un componente publica un evento que describe algo que ya pasó, como `PedidoCreado`, y otros componentes se suscriben y reaccionan, por ejemplo enviando un email o reservando stock. El que publica no sabe quién escucha, así que se pueden agregar reacciones nuevas sin modificarlo.

### Respuesta ampliada (semi-senior)

EDA reemplaza llamadas directas por eventos: hechos inmutables, en pasado, que el productor publica sin conocer a los consumidores. Distingo eventos de comandos (un destinatario, puede rechazarse) y eventos de dominio (dentro del servicio) de eventos de integración (contrato público entre servicios, vía broker). Elimina el acoplamiento de conocimiento y, con un broker, el temporal; pero el contrato del evento sigue acoplando y aparecen consistencia eventual, mensajes duplicados o desordenados y flujos más difíciles de rastrear, que manejo con consumidores idempotentes, outbox, correlation ids y trazas distribuidas. Para flujos entre servicios elijo entre coreografía y orquestación con sagas según la complejidad y la necesidad de compensaciones.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre un evento y un comando?**
El evento informa un hecho pasado a cualquiera que escuche; el comando pide una acción a un destinatario concreto, que puede rechazarla.

**2. ¿Los eventos desacoplan totalmente los servicios?**
No: eliminan el conocimiento mutuo (y con un broker, la dependencia temporal), pero todos dependen del contrato del evento.

**3. ¿Qué es la consistencia eventual?**
Que después de un cambio, las demás partes del sistema se actualizan un tiempo después, no en la misma transacción.

-----

## Práctica

**Ejercicio 1.** Clasifica cada mensaje como evento, comando o consulta, y corrige el nombre si está mal:

1. `UsuarioRegistrado`
2. `EnviarEmailDeBienvenida`
3. `ProcesarPago` (publicado a varios suscriptores)
4. `ObtenerSaldo`
5. `StockActualizar`

<details>
<summary>Solución</summary>

1. Evento. Bien nombrado.
2. Comando (lo recibe un servicio de email). Bien nombrado.
3. Comando disfrazado de evento. Si es un hecho: `PedidoListoParaCobrar` o `PedidoConfirmado`; si alguien concreto debe cobrar: comando `CobrarPedido` a un solo destinatario.
4. Consulta.
5. Ambiguo. Como evento: `StockActualizado` (en pasado); como comando: `ActualizarStock`.

</details>

**Ejercicio 2.** Diseña el evento que publica un servicio de matrículas cuando un alumno se matricula. ¿Qué campos incluyes y cuáles no?

<details>
<summary>Solución</summary>

```csharp
public record AlumnoMatriculado(
    Guid EventoId,
    DateTimeOffset OcurrioEn,
    int MatriculaId,
    int AlumnoId,
    int CursoId,
    string Periodo);
```

Incluye: identidad del evento, cuándo ocurrió y los Ids necesarios para que cada consumidor busque lo que le falte. No incluye: el objeto `Alumno` completo con su dirección y teléfono (datos personales que no todos deben recibir), ni datos que cambian (el nombre del curso podría cambiar después; el consumidor lo consulta por `CursoId` si lo necesita).

</details>

-----

## Siguiente lección

[Publish/Subscribe en memoria](02-Publish%20Subscribe%20en%20memoria.md)
