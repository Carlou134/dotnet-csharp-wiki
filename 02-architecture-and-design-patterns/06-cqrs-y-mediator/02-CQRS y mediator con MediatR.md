# CQRS y mediator con MediatR

## En una frase

CQRS separa las operaciones que **cambian** estado (commands) de las que solo **leen** (queries), y un mediator como MediatR entrega cada mensaje a su único handler; juntos dan casos de uso chicos y aislados, a cambio de más archivos y una indirección que no siempre compensa.

-----

## Antes de empezar

Conviene que ya sepas:

* DTOs y proyecciones: [DTOs y mapeo](01-DTOs%20y%20mapeo.md).
* Inyección de dependencias y lifetimes: [Host, configuración y pipeline HTTP](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* CRUD con EF Core: [CRUD en APIs con EF Core](../../04-data-access-and-transactions/01-entity-framework-core/06-CRUD%20en%20APIs%20con%20EF%20Core.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **CQRS (*Command Query Responsibility Segregation*):** separar los modelos y caminos de escritura de los de lectura.
* **Command:** mensaje que pide un cambio de estado ("crear usuario"); devuelve poco o nada.
* **Query:** mensaje que pide datos sin modificar nada ("usuarios activos").
* **Handler:** clase que ejecuta un command o una query concreta.
* **Patrón mediator:** un objeto intermedio recibe mensajes y los entrega a quien los procesa, sin que el emisor conozca al receptor.
* **Vertical slice:** organizar el código por caso de uso (carpeta `CreateUsuario/`) en lugar de por capa técnica (carpeta `Services/`).

-----

## El problema

Un `UsuarioService` empezó con cuatro métodos. Dos años después:

```csharp
public sealed class UsuarioService(
    AppDbContext db, IEmailSender email, IPasswordHasher hasher, IMapper mapper,
    IAuditoria auditoria, IStorage storage, ICache cache, ILogger<UsuarioService> logger)
{
    public Task<Guid> CrearAsync(...) { ... }
    public Task<List<UsuarioDto>> ListarAsync(...) { ... }
    public Task<UsuarioDetalleDto> ObtenerAsync(...) { ... }
    public Task CambiarPasswordAsync(...) { ... }
    public Task SubirAvatarAsync(...) { ... }
    public Task<ReporteDto> ReporteMensualAsync(...) { ... }
    // ... 19 métodos más
}
```

Lo que pasa:

1. Cada método usa dos o tres de las ocho dependencias, pero el constructor exige todas. Probar `ListarAsync` obliga a crear *mocks* de `IEmailSender` y `IStorage`.
2. Dos personas que tocan casos de uso distintos editan el mismo archivo de 1500 líneas: conflictos de *merge* cada semana.
3. `ListarAsync` reutiliza "por comodidad" el mismo método de carga que usa `CambiarPasswordAsync`, con `Include` y tracking. La lectura se vuelve lenta por una necesidad de la escritura.

Las consecuencias:

* clases que crecen sin límite y concentran todos los cambios;
* pruebas que configuran dependencias que no usan;
* lecturas penalizadas por decisiones pensadas para escribir.

-----

## Cómo funciona

### 1. CQRS: dos caminos

```text
                ┌── Command (CreateUsuario) ──► Handler ──► entidad + reglas ──► SaveChanges
Endpoint HTTP ──┤
                └── Query (GetUsuarios) ──────► Handler ──► Select a DTO, AsNoTracking
```

| | Command | Query |
| --- | --- | --- |
| Intención | Cambiar estado | Leer |
| Verbo HTTP típico | POST, PUT, PATCH, DELETE | GET |
| Devuelve | Nada, un id o un resultado mínimo | DTOs |
| Usa | Entidades, reglas de negocio, transacción | Proyecciones, sin tracking |
| Efectos secundarios | Sí | No |

**Desmitificando:** CQRS no exige dos bases de datos, ni *event sourcing*, ni MediatR. El nivel básico, que es el de esta lección, es separar el código: handlers distintos para escribir y para leer, sobre la misma base. Las variantes con base de lectura separada existen, pero agregan consistencia eventual y solo se justifican con requisitos de escala concretos.

### 2. Commands y queries como records

```csharp
public sealed record CreateUsuarioCommand(string Nombre, string Email) : IRequest<Guid>;

public sealed record GetUsuariosQuery : IRequest<IReadOnlyList<UsuarioDto>>;

public sealed record GetUsuarioPorIdQuery(Guid Id) : IRequest<UsuarioDto?>;
```

* `IRequest<TResponse>` declara qué devuelve el handler.
* Records inmutables: el mensaje es un hecho de entrada, nadie debería modificarlo en el camino.
* Un command sin resultado implementa `IRequest` (sin genérico) y su handler devuelve `Task`.

### 3. Un handler por mensaje

```csharp
public sealed class CreateUsuarioCommandHandler(UsuariosContext db)
    : IRequestHandler<CreateUsuarioCommand, Guid>
{
    public async Task<Guid> Handle(CreateUsuarioCommand request, CancellationToken cancellationToken)
    {
        var usuario = new Usuario { Nombre = request.Nombre.Trim(), Email = request.Email.Trim().ToLowerInvariant() };
        db.Usuarios.Add(usuario);
        await db.SaveChangesAsync(cancellationToken);
        return usuario.Id;
    }
}
```

El handler recibe **solo** lo que usa. Probarlo es instanciarlo con un contexto de prueba y llamar a `Handle`.

Qué va en el handler: orquestar el caso de uso (cargar, llamar al dominio, guardar). Qué **no** va: `HttpContext`, códigos de estado, `IActionResult`. El handler no sabe que existe HTTP; por eso también sirve desde un *worker* o una prueba.

### 4. MediatR: registro y envío

```csharp
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```

* `RegisterServicesFromAssembly` busca todos los `IRequestHandler<,>` del ensamblado y los registra.
* Desde MediatR 12, esto viene en el paquete `MediatR`. El paquete `MediatR.Extensions.Microsoft.DependencyInjection` quedó **obsoleto**: no lo instales.
* Desde MediatR 13 (2025), la biblioteca tiene **licencia comercial** para muchos usos y se configura con `cfg.LicenseKey`. Revisa las condiciones antes de adoptarla en un proyecto nuevo.

```csharp
app.MapPost("/usuarios", async (CreateUsuarioCommand command, ISender sender, CancellationToken ct) =>
{
    Guid id = await sender.Send(command, ct);
    return TypedResults.Created($"/usuarios/{id}", new { id });
});
```

`ISender` solo envía requests; `IMediator` además publica notificaciones. Pide la interfaz más chica que necesites. Y crear algo responde **201** con `Location`, no `200` con el id suelto.

### 5. Organización por caso de uso

```text
Features/
└── Usuarios/
    ├── CreateUsuario/
    │   ├── CreateUsuarioCommand.cs
    │   ├── CreateUsuarioCommandHandler.cs
    │   └── CreateUsuarioCommandValidator.cs
    └── GetUsuarios/
        ├── GetUsuariosQuery.cs
        └── GetUsuariosQueryHandler.cs
```

Cambiar "crear usuario" toca una carpeta. Borrar un caso de uso es borrar una carpeta.

### 6. Lo que cuesta

| Ganas | Pagas |
| --- | --- |
| Clases chicas con dependencias justas | Tres archivos por caso de uso donde antes había un método |
| Pruebas aisladas por caso de uso | **F12** en `Send(...)` te lleva a MediatR, no a tu handler |
| Un pipeline común para validación, logs y transacciones ([lección 4](04-Validacion%20con%20pipeline%20behaviors.md)) | Errores de registro (handler faltante) en tiempo de ejecución |
| Lecturas y escrituras optimizadas por separado | Una dependencia con licencia comercial |

**El mediator no desacopla tanto como parece.** El endpoint sigue dependiendo del command y de su respuesta; solo deja de conocer la clase del handler. Si lo que quieres es separar casos de uso, también puedes inyectar el handler directamente:

```csharp
builder.Services.AddScoped<CreateUsuarioCommandHandler>();

app.MapPost("/usuarios", async (CreateUsuarioCommand command, CreateUsuarioCommandHandler handler, CancellationToken ct) =>
{
    Guid id = await handler.Handle(command, ct);
    return TypedResults.Created($"/usuarios/{id}", new { id });
});
```

Sin biblioteca, con navegación directa. Lo que pierdes es el pipeline común: los *behaviors* de la lección 4.

-----

## Ejemplo completo

Consola con DI, MediatR y EF Core sobre SQLite. `Cqrs.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="MediatR" Version="13.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection();
services.AddLogging();
services.AddDbContext<UsuariosContext>(o => o.UseSqlite("Data Source=cqrs.db"));
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

await using var provider = services.BuildServiceProvider();

await using (var scope = provider.CreateAsyncScope())
{
    var db = scope.ServiceProvider.GetRequiredService<UsuariosContext>();
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();
}

// Un scope por "petición", como haría ASP.NET Core
await using (var scope = provider.CreateAsyncScope())
{
    var sender = scope.ServiceProvider.GetRequiredService<ISender>();

    Guid anaId = await sender.Send(new CreateUsuarioCommand("Ana", "Ana@Example.com"));
    await sender.Send(new CreateUsuarioCommand("Luis", "luis@example.com"));
    Console.WriteLine($"Id de Ana generado: {anaId != Guid.Empty}");

    IReadOnlyList<UsuarioDto> usuarios = await sender.Send(new GetUsuariosQuery());
    foreach (var u in usuarios)
    {
        Console.WriteLine($"  {u.Nombre} <{u.Email}>");
    }

    UsuarioDto? ana = await sender.Send(new GetUsuarioPorIdQuery(anaId));
    UsuarioDto? nadie = await sender.Send(new GetUsuarioPorIdQuery(Guid.NewGuid()));
    Console.WriteLine($"Por id: {ana?.Nombre ?? "(no existe)"} / {nadie?.Nombre ?? "(no existe)"}");
}

// ---------- Commands ----------
public sealed record CreateUsuarioCommand(string Nombre, string Email) : IRequest<Guid>;

public sealed class CreateUsuarioCommandHandler(UsuariosContext db) : IRequestHandler<CreateUsuarioCommand, Guid>
{
    public async Task<Guid> Handle(CreateUsuarioCommand request, CancellationToken cancellationToken)
    {
        var usuario = new Usuario
        {
            Nombre = request.Nombre.Trim(),
            Email = request.Email.Trim().ToLowerInvariant()
        };
        db.Usuarios.Add(usuario);
        await db.SaveChangesAsync(cancellationToken);
        return usuario.Id;
    }
}

// ---------- Queries ----------
public sealed record UsuarioDto(Guid Id, string Nombre, string Email);

public sealed record GetUsuariosQuery : IRequest<IReadOnlyList<UsuarioDto>>;

public sealed class GetUsuariosQueryHandler(UsuariosContext db) : IRequestHandler<GetUsuariosQuery, IReadOnlyList<UsuarioDto>>
{
    public async Task<IReadOnlyList<UsuarioDto>> Handle(GetUsuariosQuery request, CancellationToken cancellationToken) =>
        await db.Usuarios
            .AsNoTracking()
            .OrderBy(u => u.Nombre)
            .Select(u => new UsuarioDto(u.Id, u.Nombre, u.Email))
            .ToListAsync(cancellationToken);
}

public sealed record GetUsuarioPorIdQuery(Guid Id) : IRequest<UsuarioDto?>;

public sealed class GetUsuarioPorIdQueryHandler(UsuariosContext db) : IRequestHandler<GetUsuarioPorIdQuery, UsuarioDto?>
{
    public Task<UsuarioDto?> Handle(GetUsuarioPorIdQuery request, CancellationToken cancellationToken) =>
        db.Usuarios
            .Where(u => u.Id == request.Id)
            .Select(u => new UsuarioDto(u.Id, u.Nombre, u.Email))
            .FirstOrDefaultAsync(cancellationToken);
}

// ---------- Persistencia ----------
public class Usuario
{
    public Guid Id { get; set; }
    public required string Nombre { get; set; }
    public required string Email { get; set; }
}

public sealed class UsuariosContext(DbContextOptions<UsuariosContext> options) : DbContext(options)
{
    public DbSet<Usuario> Usuarios => Set<Usuario>();

    protected override void OnModelCreating(ModelBuilder modelBuilder) =>
        modelBuilder.Entity<Usuario>().HasIndex(u => u.Email).IsUnique();
}
```

Salida:

```text
Id de Ana generado: True
  Ana <ana@example.com>
  Luis <luis@example.com>
Por id: Ana / (no existe)
```

Qué observar:

* El código de `Program.cs` no conoce ninguna clase handler: solo commands, queries y `ISender`.
* El command normalizó el email (`Ana@Example.com` → `ana@example.com`): la regla vive en un solo lugar.
* Las queries proyectan a DTOs sin tracking; el command trabaja con la entidad. Cada camino usa lo que le conviene.
* `GetUsuarioPorIdQuery` devuelve `null` si no existe; traducir eso a `404` es trabajo del endpoint, no del handler.
* `services.AddLogging()` está porque MediatR usa `ILogger` internamente; ASP.NET Core ya lo registra por ti.

-----

## Errores comunes

**1. Instalar `MediatR.Extensions.Microsoft.DependencyInjection`.**
Qué pasa: un paquete obsoleto que puede arrastrar una versión vieja de MediatR o duplicar registros.
Por qué: desde MediatR 12, `AddMediatR` viene en el paquete principal.
Arreglo: solo `MediatR`, con `AddMediatR(cfg => cfg.RegisterServicesFromAssembly(...))`.

**2. Queries que modifican estado.**
Qué pasa: un `GetUsuarioQuery` que además actualiza `UltimoAcceso`; cachear o reintentar la query cambia datos.
Por qué: se rompió la única regla de CQRS.
Arreglo: la query solo lee; el cambio va en un command explícito.

**3. Handlers que devuelven `IActionResult` o leen `HttpContext`.**
Qué pasa: el caso de uso no se puede usar desde un worker ni probar sin un servidor.
Por qué: la capa de aplicación quedó atada a HTTP.
Arreglo: el handler devuelve datos o un resultado de dominio; el endpoint decide el código HTTP.

**4. Un handler que llama a `Send` de otro handler.**
Qué pasa: cadenas de handlers difíciles de seguir, transacciones anidadas y pipelines que se ejecutan dos veces.
Por qué: se usa el mediator como un bus de servicios.
Arreglo: extrae la lógica compartida a un servicio o al dominio y llámalo desde ambos handlers.

**5. Creer que MediatR "es" CQRS o que CQRS exige dos bases.**
Qué pasa: se rechaza CQRS por miedo a la complejidad, o se adopta MediatR esperando beneficios de arquitectura que no da solo.
Por qué: se confunde el patrón con una herramienta y con su variante más compleja.
Arreglo: CQRS es separar lectura y escritura; MediatR es una forma de despachar mensajes.

-----

## Según la versión de .NET

* **C# 9:** records, ideales para commands y queries.
* **C# 12:** constructores primarios en clases: handlers con dependencias en una línea.
* **MediatR 12 (2023):** registro en DI integrado; desaparece el paquete de extensiones.
* **MediatR 12.5 (2025):** `RequestHandlerDelegate` recibe el `CancellationToken`.
* **MediatR 13 (2025):** licencia comercial con `LicenseKey`.

-----

## Cuándo sí y cuándo no

**Usa commands y queries separados cuando:** la aplicación tiene muchos casos de uso con dependencias distintas, las lecturas y escrituras tienen necesidades diferentes o varios equipos trabajan en paralelo.

**Usa MediatR cuando:** además quieres un pipeline común (validación, logs, transacciones) y aceptas su licencia.

**No lo uses cuando:** es un CRUD chico. Un endpoint con un `Select` o un `SaveChangesAsync` es más claro que tres archivos y un mediator.

-----

## Resumen en 5 líneas

1. Commands cambian estado y devuelven poco; queries leen y no cambian nada.
2. Un handler por mensaje, con solo las dependencias que usa.
3. CQRS no exige dos bases ni MediatR: el nivel básico es separar el código.
4. MediatR 12+ trae el registro en DI y desde la v13 tiene licencia comercial.
5. El handler no conoce HTTP; el endpoint traduce el resultado a códigos de estado.

-----

## Para profundizar

<details>
<summary>Alternativas a MediatR</summary>

* **Inyectar el handler directamente:** sin biblioteca; pierdes el pipeline común, que puedes reemplazar con decoradores (Scrutor) o *endpoint filters*.
* **Mediator (martinothamar):** misma API conceptual, generada en compilación; sin reflexión y compatible con Native AOT.
* **Wolverine:** mediator y bus de mensajería en uno, con outbox integrado.

La pregunta útil no es "¿qué mediator?", sino "¿qué problema me resuelve el mediator que no resuelva una clase con un método?".

</details>

<details>
<summary>Commands que devuelven datos</summary>

La versión estricta de CQRS dice que un command no devuelve nada. En APIs es práctico devolver el id generado, o un resultado de éxito o error, para responder `201` con `Location`. Lo que no conviene es que un command devuelva la entidad completa: estarías mezclando de nuevo lectura y escritura.

</details>

-----

## En entrevista

### Respuesta corta (junior)

CQRS separa commands, que cambian datos, de queries, que solo leen. Con MediatR, cada command o query es un `IRequest` y tiene su `IRequestHandler`. El controlador o endpoint solo hace `Send` y no conoce la lógica.

### Respuesta ampliada (semi-senior)

Uso CQRS a nivel de código: commands con entidades y reglas, queries con proyecciones sin tracking, sobre la misma base. Organizo por caso de uso (*vertical slices*) y cada handler recibe solo sus dependencias. MediatR me da el despacho y un pipeline de *behaviors* para validación o logs, pero tiene costos: indirección, errores de registro en ejecución y licencia comercial desde la v13. En un CRUD chico no lo uso; a veces basta con inyectar el handler. El handler no conoce HTTP; el endpoint traduce a `201`, `404` o `400`.

### Preguntas frecuentes de seguimiento

**1. ¿CQRS requiere dos bases de datos?**
No. Esa es una variante avanzada con consistencia eventual; la separación de código es suficiente en la mayoría de los casos.

**2. ¿`ISender` o `IMediator`?**
`ISender` si solo envías requests; `IMediator` incluye también `Publish`.

**3. ¿Puede un handler llamar a otro con `Send`?**
Técnicamente sí, pero complica el flujo; mejor extraer la lógica compartida.

**4. ¿Qué lifetime tienen los handlers?**
Transient por defecto; pueden recibir servicios scoped como el `DbContext` porque se resuelven dentro del scope de la petición.

-----

## Práctica

**Ejercicio 1.** Agrega el command `DesactivarUsuarioCommand(Guid Id)` que no devuelve datos. Si el usuario no existe, el endpoint debe responder `404`. ¿Qué devuelve el handler?

<details>
<summary>Solución</summary>

El handler devuelve un `bool` (o un resultado de dominio), no un código HTTP:

```csharp
public sealed record DesactivarUsuarioCommand(Guid Id) : IRequest<bool>;

public sealed class DesactivarUsuarioCommandHandler(UsuariosContext db) : IRequestHandler<DesactivarUsuarioCommand, bool>
{
    public async Task<bool> Handle(DesactivarUsuarioCommand request, CancellationToken cancellationToken)
    {
        int filas = await db.Usuarios
            .Where(u => u.Id == request.Id)
            .ExecuteUpdateAsync(s => s.SetProperty(u => u.Activo, false), cancellationToken);
        return filas > 0;
    }
}

app.MapPost("/usuarios/{id:guid}/desactivar", async (Guid id, ISender sender, CancellationToken ct) =>
    await sender.Send(new DesactivarUsuarioCommand(id), ct) ? Results.NoContent() : Results.NotFound());
```

(Requiere agregar `public bool Activo { get; set; } = true;` a `Usuario`.)

</details>

**Ejercicio 2.** Este query handler tiene un problema de CQRS y uno de rendimiento. Encuéntralos.

```csharp
public async Task<UsuarioDto> Handle(GetUsuarioQuery request, CancellationToken ct)
{
    var usuario = await db.Usuarios.Include(u => u.Pedidos).SingleAsync(u => u.Id == request.Id, ct);
    usuario.UltimaConsulta = DateTime.UtcNow;
    await db.SaveChangesAsync(ct);
    return new UsuarioDto(usuario.Id, usuario.Nombre, usuario.Email);
}
```

<details>
<summary>Solución</summary>

* **CQRS:** la query modifica estado (`UltimaConsulta` + `SaveChangesAsync`). Si eso es necesario, va en un command aparte o en un mecanismo de auditoría.
* **Rendimiento:** carga todos los pedidos con `Include` y los rastrea, para devolver tres campos del usuario.
* **Extra:** `SingleAsync` lanza excepción si no existe; la query debería devolver `null` y dejar que el endpoint responda `404`.

```csharp
public Task<UsuarioDto?> Handle(GetUsuarioQuery request, CancellationToken ct) =>
    db.Usuarios
        .Where(u => u.Id == request.Id)
        .Select(u => new UsuarioDto(u.Id, u.Nombre, u.Email))
        .FirstOrDefaultAsync(ct);
```

</details>

-----

## Siguiente lección

[Notificaciones en proceso con INotification](03-Notificaciones%20en%20proceso%20con%20INotification.md)
