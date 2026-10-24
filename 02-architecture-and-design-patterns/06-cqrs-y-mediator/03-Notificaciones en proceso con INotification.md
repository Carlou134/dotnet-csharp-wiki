# Notificaciones en proceso con INotification

## En una frase

`INotification` anuncia que algo ya pasó y deja que cero o más handlers reaccionen dentro del mismo proceso; pero `Publish` **espera** a los handlers y, por defecto, **propaga sus excepciones**, así que una notificación no es ni asíncrona ni aislada ni confiable salvo que la diseñes para serlo.

-----

## Antes de empezar

Conviene que ya sepas:

* Commands, handlers y MediatR: [CQRS y mediator con MediatR](02-CQRS%20y%20mediator%20con%20MediatR.md).
* Eventos frente a comandos y bus en memoria: [Publish/Subscribe en memoria](../02-event-driven-architecture/02-Publish%20Subscribe%20en%20memoria.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Notificación:** mensaje que comunica un hecho pasado ("usuario creado") y puede tener cero o muchos handlers.
* **Publicador de notificaciones (*notification publisher*):** estrategia que decide cómo se ejecutan los handlers de una notificación: en secuencia, en paralelo, aislando errores.
* **Efecto secundario:** trabajo que ocurre como consecuencia de un caso de uso pero no es su objetivo (email, auditoría, caché).
* **Outbox:** tabla donde se guarda el evento en la misma transacción que el cambio, para publicarlo después de forma confiable.

-----

## El problema

El command de registro publica una notificación para mandar el email de bienvenida:

```csharp
db.Usuarios.Add(usuario);
await db.SaveChangesAsync(ct);
await publisher.Publish(new UsuarioCreadoNotification(usuario.Id, usuario.Email), ct);
return usuario.Id;
```

Un día el servidor SMTP está caído:

1. `SaveChangesAsync` ya guardó al usuario.
2. El handler del email lanza `InvalidOperationException: SMTP no disponible`.
3. `Publish` propaga la excepción, el command falla y el endpoint responde **500**.
4. El usuario ve "error", reintenta el registro y ahora recibe "**el correo ya está registrado**". Su cuenta existe, pero él no lo sabe y nunca recibió el email.

Y en un día normal: el handler del email tarda 3 segundos en hablar con el proveedor. El registro, que tardaba 40 ms, ahora tarda 3 segundos, porque `Publish` espera a todos los handlers.

Las consecuencias:

* un efecto secundario que falla convierte un caso de uso exitoso en un error;
* reintentos del cliente que chocan con datos que sí se guardaron;
* latencia de cualquier handler sumada a la petición.

-----

## Cómo funciona

### 1. `IRequest` frente a `INotification`

| | `IRequest<T>` | `INotification` |
| --- | --- | --- |
| Significa | "Haz esto" | "Esto ya pasó" |
| Handlers | Exactamente uno (si falta, excepción) | Cero, uno o muchos (si no hay, no pasa nada) |
| Se envía con | `Send` | `Publish` |
| Devuelve | `T` | `Task` (nada) |
| ¿El emisor espera? | Sí | **Sí**: `Publish` termina cuando terminan los handlers |
| ¿Errores al emisor? | Sí | **Sí**, con el publicador por defecto |

La segunda mitad de la tabla es la que suele explicarse mal: "no espera respuesta" no significa "no espera".

### 2. Notificación y handlers

```csharp
public sealed record UsuarioCreadoNotification(Guid UsuarioId, string Email) : INotification;

public sealed class EnviarEmailBienvenidaHandler(IEmailSender email)
    : INotificationHandler<UsuarioCreadoNotification>
{
    public Task Handle(UsuarioCreadoNotification notification, CancellationToken cancellationToken) =>
        email.EnviarAsync(notification.Email, "Bienvenido", cancellationToken);
}
```

* Nombre en pasado: `UsuarioCreado`, no `CrearUsuario`.
* Solo datos del hecho; los handlers consultan lo demás si lo necesitan.
* `RegisterServicesFromAssembly` registra todos los `INotificationHandler<>` automáticamente.

### 3. Cómo ejecuta los handlers MediatR

```text
ForeachAwaitPublisher (por defecto)        TaskWhenAllPublisher
handler A ──► termina                      handler A ─┐
handler B ──► lanza ✖                      handler B ─┼─► espera a todos ──► lanza si alguno falló
handler C ──► NO se ejecuta                handler C ─┘
excepción ──► llega al command             excepción ──► llega al command
```

| Publicador | Orden | Si uno falla | Cuidado |
| --- | --- | --- | --- |
| `ForeachAwaitPublisher` | Secuencial | Los siguientes no corren; la excepción sube | Latencia = suma de todos |
| `TaskWhenAllPublisher` | Concurrente | Todos corren; la excepción sube al final | Handlers que comparten el `DbContext` scoped chocan: no es seguro entre hilos |
| Propio (`INotificationPublisher`) | El que programes | El que programes | Tú decides qué se registra y qué se propaga |

Se elige en el registro: `cfg.NotificationPublisherType = typeof(TaskWhenAllPublisher);`. El **orden** entre handlers no está garantizado: no escribas un handler que dependa de que otro corrió antes.

### 4. Aislar los errores de los efectos secundarios

```csharp
public sealed class PublicadorTolerante(ILogger<PublicadorTolerante> logger) : INotificationPublisher
{
    public async Task Publish(
        IEnumerable<NotificationHandlerExecutor> handlerExecutors,
        INotification notification,
        CancellationToken cancellationToken)
    {
        foreach (var executor in handlerExecutors)
        {
            try
            {
                await executor.HandlerCallback(notification, cancellationToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Falló {Handler}", executor.HandlerInstance.GetType().Name);
            }
        }
    }
}
```

Cada handler corre aunque otro haya fallado, y el command no se entera. El filtro `when` deja pasar la cancelación: si el cliente se fue, no tiene sentido seguir.

Esto resuelve el 500 del problema, pero el email **sigue perdido**: solo quedó un log.

### 5. En proceso no es confiable

```text
SaveChangesAsync ✔ ──► Publish ──► handler email ✖ (o el proceso se reinicia aquí)
                                    │
                                    └─► el usuario existe, el email no se envió y nada lo reintentará
```

Una notificación en memoria vive lo que vive el proceso. Si el efecto secundario **importa** (cobrar, avisar a otro sistema, emitir una factura), necesitas persistirlo:

* **Outbox:** guarda el evento en una tabla dentro de la misma transacción que el usuario; un proceso en segundo plano lo lee y lo entrega con reintentos. Ver [Entrega confiable e idempotencia](../02-event-driven-architecture/03-Entrega%20confiable%20e%20idempotencia.md).
* **Cola de trabajos:** encolar el envío del email (por ejemplo, con un `Channel<T>` y un `BackgroundService`, o una cola real) en lugar de ejecutarlo dentro de la petición.

### 6. ¿Publicar antes o después de guardar?

| Momento | Ventaja | Riesgo |
| --- | --- | --- |
| Después de `SaveChangesAsync` | Los handlers ven datos confirmados | Si un handler falla, el cambio ya está guardado |
| Antes de `SaveChangesAsync`, mismo `DbContext` | Los cambios de los handlers entran en la misma transacción | Un handler lento alarga la transacción; efectos externos (email) no se pueden deshacer |

Regla práctica: lo que modifica **la misma base** puede ir antes, dentro de la transacción. Lo que sale **afuera** (emails, HTTP) va después, y si es importante, por outbox.

-----

## Ejemplo completo

Consola que muestra qué pasa cuando un handler falla, con el publicador por defecto y con uno tolerante. Mismo `.csproj` que la [lección 2](02-CQRS%20y%20mediator%20con%20MediatR.md) (`MediatR` 13.0.0 y `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0).

`Program.cs`:

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

await Ejecutar(tolerante: false);
await Ejecutar(tolerante: true);

static async Task Ejecutar(bool tolerante)
{
    var services = new ServiceCollection();
    services.AddLogging();
    services.AddSingleton<Registro>();
    services.AddDbContext<UsuariosContext>(o => o.UseSqlite("Data Source=notificaciones.db"));
    services.AddMediatR(cfg =>
    {
        cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
        if (tolerante)
        {
            cfg.NotificationPublisherType = typeof(PublicadorTolerante);
        }
    });

    await using var provider = services.BuildServiceProvider();
    await using var scope = provider.CreateAsyncScope();

    var db = scope.ServiceProvider.GetRequiredService<UsuariosContext>();
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();

    Console.WriteLine(tolerante ? "-- Publicador tolerante --" : "-- Publicador por defecto --");
    var sender = scope.ServiceProvider.GetRequiredService<ISender>();
    try
    {
        await sender.Send(new CreateUsuarioCommand("ana@example.com"));
        Console.WriteLine("Send terminó bien");
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"Send lanzó: {ex.Message}");
    }

    Console.WriteLine($"Usuarios guardados: {await db.Usuarios.CountAsync()}");

    if (tolerante)
    {
        foreach (var linea in scope.ServiceProvider.GetRequiredService<Registro>().Lineas.Order())
        {
            Console.WriteLine($"  {linea}");
        }
    }
}

// ---------- Command ----------
public sealed record CreateUsuarioCommand(string Email) : IRequest<Guid>;

public sealed class CreateUsuarioCommandHandler(UsuariosContext db, IPublisher publisher)
    : IRequestHandler<CreateUsuarioCommand, Guid>
{
    public async Task<Guid> Handle(CreateUsuarioCommand request, CancellationToken cancellationToken)
    {
        var usuario = new Usuario { Email = request.Email };
        db.Usuarios.Add(usuario);
        await db.SaveChangesAsync(cancellationToken);

        await publisher.Publish(new UsuarioCreadoNotification(usuario.Id, usuario.Email), cancellationToken);
        return usuario.Id;
    }
}

// ---------- Notificación y handlers ----------
public sealed record UsuarioCreadoNotification(Guid UsuarioId, string Email) : INotification;

public sealed class AuditarUsuarioCreadoHandler(Registro registro) : INotificationHandler<UsuarioCreadoNotification>
{
    public Task Handle(UsuarioCreadoNotification notification, CancellationToken cancellationToken)
    {
        registro.Lineas.Add($"auditoría: {notification.Email}");
        return Task.CompletedTask;
    }
}

public sealed class EnviarEmailBienvenidaHandler : INotificationHandler<UsuarioCreadoNotification>
{
    public Task Handle(UsuarioCreadoNotification notification, CancellationToken cancellationToken) =>
        Task.FromException(new InvalidOperationException("SMTP no disponible"));
}

// ---------- Publicador que aísla errores ----------
public sealed class PublicadorTolerante(Registro registro) : INotificationPublisher
{
    public async Task Publish(
        IEnumerable<NotificationHandlerExecutor> handlerExecutors,
        INotification notification,
        CancellationToken cancellationToken)
    {
        foreach (var executor in handlerExecutors)
        {
            try
            {
                await executor.HandlerCallback(notification, cancellationToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                registro.Lineas.Add($"error aislado en {executor.HandlerInstance.GetType().Name}: {ex.Message}");
            }
        }
    }
}

// Solo para el ejemplo: no es seguro entre hilos
public sealed class Registro
{
    public List<string> Lineas { get; } = [];
}

// ---------- Persistencia ----------
public class Usuario
{
    public Guid Id { get; set; }
    public required string Email { get; set; }
}

public sealed class UsuariosContext(DbContextOptions<UsuariosContext> options) : DbContext(options)
{
    public DbSet<Usuario> Usuarios => Set<Usuario>();
}
```

Salida:

```text
-- Publicador por defecto --
Send lanzó: SMTP no disponible
Usuarios guardados: 1
-- Publicador tolerante --
Send terminó bien
Usuarios guardados: 1
  auditoría: ana@example.com
  error aislado en EnviarEmailBienvenidaHandler: SMTP no disponible
```

Qué observar:

* Con el publicador por defecto, `Send` **falla** pero el usuario **quedó guardado**: el cliente recibiría un error por algo que sí ocurrió.
* Con el publicador tolerante, el command termina bien, la auditoría corre y el fallo del email queda registrado.
* El email sigue sin enviarse en ambos casos. Aislar el error no lo reintenta: para eso hace falta persistir el evento (outbox) o encolar el trabajo.
* El registro se imprime ordenado porque MediatR no garantiza el orden de los handlers.

-----

## Errores comunes

**1. Creer que `Publish` es "dispara y olvida".**
Qué pasa: la latencia de todos los handlers se suma a la petición, y sus errores llegan al command.
Por qué: `Publish` devuelve un `Task` que termina cuando terminan los handlers.
Arreglo: mide; si un efecto es lento o externo, encólalo en lugar de ejecutarlo en la petición.

**2. Suponer que un handler que falla "no rompe el command".**
Qué pasa: `500` con los datos ya guardados; reintentos que chocan con duplicados.
Por qué: el publicador por defecto propaga la primera excepción.
Arreglo: publicador propio que aísle errores, o mover el efecto fuera de la petición (outbox, cola).

**3. Ejecutar los handlers en paralelo compartiendo el `DbContext`.**
Qué pasa: `InvalidOperationException: A second operation was started on this context instance...`.
Por qué: `TaskWhenAllPublisher` corre los handlers a la vez y todos reciben la misma instancia scoped.
Arreglo: publicador secuencial, o un contexto por handler con `IDbContextFactory<T>`.

**4. Depender del orden de los handlers.**
Qué pasa: funciona hasta que alguien agrega un handler o reordena archivos.
Por qué: el orden sale del registro por reflexión, no de una regla.
Arreglo: handlers independientes; si hay una secuencia obligatoria, es un solo caso de uso.

**5. Usar notificaciones para la lógica principal.**
Qué pasa: el "descontar stock" vive en un handler de `PedidoCreado` y nadie sabe que el pedido depende de él.
Por qué: la notificación oculta pasos esenciales detrás de un `Publish`.
Arreglo: lo esencial va en el command (o en el dominio); las notificaciones, para efectos secundarios.

-----

## Según la versión de .NET

* **C# 9:** records: notificaciones inmutables en una línea.
* **C# 6:** filtros de excepción (`catch ... when`), usados para dejar pasar la cancelación.
* **MediatR 12 (2023):** `INotificationPublisher` reemplaza la estrategia de publicación que antes se sobrescribía heredando de `Mediator`; vienen `ForeachAwaitPublisher` y `TaskWhenAllPublisher`.
* **MediatR 13 (2025):** licencia comercial.

-----

## Cuándo sí y cuándo no

**Usa `INotification` cuando:** varios efectos secundarios **en el mismo proceso** reaccionan a un hecho y su fallo no debe invalidar el caso de uso (auditoría, invalidar caché, métricas).

**Usa outbox o una cola cuando:** el efecto es importante, sale del proceso o debe reintentarse (emails, facturas, integración con otros sistemas).

**No uses notificaciones cuando:** el paso es parte esencial del caso de uso, o necesitas un resultado del handler.

-----

## Resumen en 5 líneas

1. `INotification` comunica un hecho pasado a cero o más handlers con `Publish`.
2. `Publish` espera a los handlers y, por defecto, propaga la primera excepción.
3. Un `INotificationPublisher` propio puede aislar errores; `TaskWhenAllPublisher` corre en paralelo con sus riesgos.
4. En proceso no es confiable: si el efecto importa, outbox o cola.
5. Lo esencial va en el command; las notificaciones, para efectos secundarios.

-----

## Para profundizar

<details>
<summary>Eventos de dominio despachados al guardar</summary>

Un patrón común es que las entidades acumulen eventos (`usuario.AgregarEvento(new UsuarioCreado(...))`) y que un `SaveChangesInterceptor` los publique. Si se publican **antes** de confirmar (en `SavingChangesAsync`), los handlers pueden modificar la misma base dentro de la transacción. Si se publican **después** (en `SavedChangesAsync`), los handlers ven datos confirmados, pero un fallo ya no deshace nada. Los eventos de integración (hacia otros sistemas) se escriben en el outbox dentro de la transacción.

</details>

<details>
<summary>Notificaciones de MediatR frente a un bus en memoria propio</summary>

La [lección de Pub/Sub en memoria](../02-event-driven-architecture/02-Publish%20Subscribe%20en%20memoria.md) construye un bus que resuelve handlers desde el contenedor, aísla errores y permite suscripciones dinámicas. MediatR resuelve lo mismo con menos código propio, pero con su licencia y sus decisiones por defecto (secuencial, propaga errores). Para efectos en proceso, cualquiera de los dos sirve; para entrega confiable, ninguno alcanza solo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`INotification` sirve para avisar que algo pasó, como "usuario creado", y puede tener varios handlers: email, auditoría, caché. Se dispara con `Publish`. A diferencia de `IRequest`, no devuelve nada y puede no tener handlers.

### Respuesta ampliada (semi-senior)

Uso notificaciones para efectos secundarios en proceso. Sé que `Publish` espera a los handlers y que el publicador por defecto es secuencial y propaga la primera excepción, así que un email caído puede devolver 500 con los datos ya guardados. Lo resuelvo con un `INotificationPublisher` que aísla errores o, si el efecto importa, con outbox y un procesador en segundo plano. Evito `TaskWhenAllPublisher` si los handlers comparten el `DbContext`, no dependo del orden de los handlers y no pongo lógica esencial en notificaciones.

### Preguntas frecuentes de seguimiento

**1. ¿Qué pasa si una notificación no tiene handlers?**
Nada: `Publish` termina sin hacer nada y sin error.

**2. ¿Los handlers corren en paralelo?**
Por defecto no, en secuencia. Con `TaskWhenAllPublisher` sí.

**3. ¿Cómo evitas que un handler que falla rompa el command?**
Con un publicador que capture y registre cada error, o sacando el efecto de la petición.

**4. ¿Una notificación garantiza que el email se envíe?**
No. Si el proceso cae o el handler falla, se pierde. Para garantías, outbox.

-----

## Práctica

**Ejercicio 1.** Agrega un handler `InvalidarCacheUsuariosHandler` que borre la clave `"usuarios"` de `IMemoryCache` cuando se crea un usuario. ¿Qué pasa si `IMemoryCache` no está registrado?

<details>
<summary>Solución</summary>

```csharp
public sealed class InvalidarCacheUsuariosHandler(IMemoryCache cache) : INotificationHandler<UsuarioCreadoNotification>
{
    public Task Handle(UsuarioCreadoNotification notification, CancellationToken cancellationToken)
    {
        cache.Remove("usuarios");
        return Task.CompletedTask;
    }
}
```

Requiere `services.AddMemoryCache()` (en ASP.NET Core suele estar registrado). Si falta, el contenedor no puede crear el handler y `Publish` lanza `InvalidOperationException` al resolverlo, **después** de que el usuario se guardó: el mismo problema de la lección, causado por configuración. Por eso conviene una prueba que publique cada notificación al menos una vez.

</details>

**Ejercicio 2.** El equipo quiere que el registro no espere al email. Proponen esto en el handler. ¿Qué está mal?

```csharp
public Task Handle(UsuarioCreadoNotification n, CancellationToken ct)
{
    _ = Task.Run(() => email.EnviarAsync(n.Email, "Bienvenido", ct));
    return Task.CompletedTask;
}
```

<details>
<summary>Solución</summary>

* Si `EnviarAsync` falla, la excepción queda en un `Task` que nadie observa: se pierde sin log.
* `email` puede depender de servicios scoped que se descartan al terminar la petición: el envío corre con dependencias ya liberadas.
* `ct` es el token de la petición: se cancela cuando la petición termina, justo cuando el envío empieza.
* Si el proceso se reinicia, el envío desaparece.

La alternativa es encolar el trabajo (un `Channel<T>` leído por un `BackgroundService`, con su propio scope y reintentos) o, si debe ser confiable, guardar el evento en un outbox.

</details>

-----

## Siguiente lección

[Validación con pipeline behaviors](04-Validacion%20con%20pipeline%20behaviors.md)
