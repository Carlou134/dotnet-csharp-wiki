# Validación con pipeline behaviors

## En una frase

Un *pipeline behavior* de MediatR envuelve a todos los handlers como un middleware; usado con FluentValidation, valida cada command **antes** de que su handler corra, y un `IExceptionHandler` traduce el fallo a un `400` con el formato estándar de ProblemDetails.

-----

## Antes de empezar

Conviene que ya sepas:

* Commands, handlers y MediatR: [CQRS y mediator con MediatR](02-CQRS%20y%20mediator%20con%20MediatR.md).
* Middleware y pipeline HTTP: [Host, configuración y pipeline HTTP](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* Formato de errores: [Errores con ProblemDetails](../../03-aspnet-core-apis/01-diseno-de-apis-rest/02-Errores%20con%20ProblemDetails.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Pipeline behavior:** clase que implementa `IPipelineBehavior<TRequest, TResponse>` y se ejecuta alrededor de cada handler de MediatR.
* **FluentValidation:** biblioteca para declarar reglas de validación con código (`RuleFor(x => x.Email).EmailAddress()`).
* **Validator:** clase `AbstractValidator<T>` con las reglas de un tipo concreto.
* **`IExceptionHandler`:** interfaz de ASP.NET Core (.NET 8+) para convertir excepciones en respuestas HTTP desde el middleware de excepciones.
* **Condición de carrera:** resultado que depende de qué operación concurrente llega primero.

-----

## El problema

Una guía de clase propone este validator y este behavior:

```csharp
public CreateUsuarioCommandValidator(UsersContext context)
{
    RuleFor(x => x.Email)
        .MustAsync(email => !context.Usuario.AnyAsync(u => u.Email == email));
}
```

```csharp
var validationResults = await Task.WhenAll(
    _validators.Select(v => v.ValidateAsync(context, cancellationToken)));
```

Y un `IExceptionFilter` de MVC que devuelve `{ "message": "Errores de validación", "errors": [...] }`.

Lo que pasa:

1. **No compila.** `!` sobre un `Task<bool>` da `CS0023: Operator '!' cannot be applied to operand of type 'Task<bool>'`. Además, `MustAsync` espera una lambda `(valor, CancellationToken)`, no de un parámetro.
2. **Falla bajo carga.** Arreglado el validator, un command con dos validators que consultan la base ejecuta ambos a la vez con `Task.WhenAll`. Los dos reciben el **mismo** `DbContext` scoped: `InvalidOperationException: A second operation was started on this context instance before a previous operation completed`.
3. **Duplicados igual.** Dos registros simultáneos con el mismo email pasan los dos el `AnyAsync` (ninguno existe todavía) y los dos se insertan.
4. **El filtro no cubre todo.** Un `IExceptionFilter` solo actúa en acciones de MVC. Un endpoint de Minimal API que usa el mismo command devuelve `500`. Y el formato inventado no es el que el resto de la API (y los clientes) esperan.

Las consecuencias:

* código de la guía que no compila tal cual;
* errores intermitentes que dependen de cuántos validators tiene un command;
* una regla de unicidad que no protege nada bajo concurrencia;
* respuestas de error con formatos distintos según el tipo de endpoint.

-----

## Cómo funciona

### 1. El pipeline de MediatR

```text
sender.Send(command)
   │
   ▼
┌─────────────────────────────┐
│ LoggingBehavior             │  antes ─┐
│  ┌───────────────────────┐  │         │
│  │ ValidationBehavior    │  │  antes ─┤  si falla: lanza, el handler NO corre
│  │  ┌─────────────────┐  │  │         │
│  │  │ Handler         │  │  │         │
│  │  └─────────────────┘  │  │         │
│  └───────────────────────┘  │  después┘
└─────────────────────────────┘
```

Cada behavior recibe el request y un delegado `next` que ejecuta el resto del pipeline. Es la misma idea que el middleware de ASP.NET Core, pero alrededor de los handlers.

### 2. El validator

```csharp
public sealed class CreateUsuarioCommandValidator : AbstractValidator<CreateUsuarioCommand>
{
    public CreateUsuarioCommandValidator(UsuariosContext db)
    {
        RuleFor(x => x.Nombre)
            .NotEmpty().WithMessage("El nombre es obligatorio.")
            .MaximumLength(100);

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress().WithMessage("El correo no es válido.")
            .MustAsync(async (email, ct) => !await db.Usuarios.AnyAsync(u => u.Email == email, ct))
            .WithMessage("El correo ya está registrado.");
    }
}
```

* `MustAsync` recibe `(valor, CancellationToken)` y devuelve `Task<bool>`. El `!` va **después** del `await`.
* `WithMessage` se aplica a la regla inmediatamente anterior.
* El validator puede recibir dependencias por constructor: `AddValidatorsFromAssembly` lo registra en DI (como *scoped*, por defecto).

### 3. El behavior, en secuencia

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = new List<ValidationFailure>();

        // En secuencia: los validators pueden compartir el DbContext scoped
        foreach (var validator in validators)
        {
            var result = await validator.ValidateAsync(context, cancellationToken);
            failures.AddRange(result.Errors);
        }

        if (failures.Count > 0)
        {
            throw new ValidationException(failures);
        }

        return await next(cancellationToken);
    }
}
```

* `IEnumerable<IValidator<TRequest>>` llega vacío si el request no tiene validators: el `foreach` no hace nada y se llama a `next`.
* `where TRequest : notnull` es la restricción de la interfaz. Restringir a `IRequest<TResponse>` excluye a los commands sin resultado (`IRequest`), que en silencio quedarían sin validar.
* El `Task.WhenAll` de la guía no aporta velocidad real (las consultas a la base no pueden ir en paralelo sobre el mismo contexto) y sí agrega el error de concurrencia.

Registro:

```csharp
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

Los behaviors se ejecutan en el orden en que se registran: el primero registrado es el más externo.

### 4. De excepción a HTTP 400 con `IExceptionHandler`

```csharp
public sealed class ValidationExceptionHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not ValidationException validation)
        {
            return false;   // no es mío: que lo maneje el siguiente
        }

        var errors = validation.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

        httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = new HttpValidationProblemDetails(errors),
            Exception = exception
        });
    }
}

builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
app.UseExceptionHandler();
```

| | `IExceptionFilter` (MVC) | `IExceptionHandler` (.NET 8+) |
| --- | --- | --- |
| Cubre | Solo acciones de controladores | Toda la aplicación: controladores, Minimal APIs, middleware |
| Formato | El que escribas | `HttpValidationProblemDetails` (RFC 9457), igual que la validación automática de `[ApiController]` |
| Encadenable | Varios filtros | Varios handlers en orden de registro; el primero que devuelve `true` gana |

### 5. Tres niveles de validación

```text
1. Forma de la entrada     validator / DataAnnotations   "¿el email tiene formato de email?"
2. Reglas de negocio       dominio / handler             "¿un pedido cerrado puede editarse?"
3. Integridad              base de datos                 índice único, FK, NOT NULL
```

La guía decía "el handler NO valida". Es cierto para el formato de la entrada; **no** para las invariantes del dominio, que deben vivir en las entidades aunque el command llegue por otro camino. Y la base es la última defensa: **el `AnyAsync` del validator mejora el mensaje, pero solo el índice único garantiza la regla**.

```csharp
modelBuilder.Entity<Usuario>().HasIndex(u => u.Email).IsUnique();
```

Entre el `AnyAsync` y el `INSERT` hay una ventana. Si dos peticiones la atraviesan juntas, una falla con `DbUpdateException` por el índice; el handler puede capturarla y convertirla en el mismo error de validación.

### 6. ¿Excepciones o resultados?

Lanzar `ValidationException` es simple y se integra con el pipeline. El costo: usar excepciones para un flujo esperado (un dato inválido no es excepcional) y que la firma del handler no diga que puede fallar. La alternativa es un tipo resultado (`Result<T>` con éxito o errores) que el behavior devuelve sin lanzar. Es más explícito y más código. Ambas son válidas si el equipo es consistente.

-----

## Ejemplo completo

Proyecto `dotnet new web`. `Validacion.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="12.0.0" />
    <PackageReference Include="MediatR" Version="13.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`FluentValidation.DependencyInjectionExtensions` trae `FluentValidation` como dependencia.

`Program.cs`:

```csharp
using FluentValidation;
using FluentValidation.Results;
using MediatR;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<UsuariosContext>(o => o.UseSqlite("Data Source=validacion.db"));
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();

var app = builder.Build();

// Solo para el ejemplo: recrea la base. En una aplicación real se usan migraciones.
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<UsuariosContext>();
    db.Database.EnsureDeleted();
    db.Database.EnsureCreated();
}

app.UseExceptionHandler();

app.MapPost("/usuarios", async (CreateUsuarioCommand command, ISender sender, CancellationToken ct) =>
{
    Guid id = await sender.Send(command, ct);
    return TypedResults.Created($"/usuarios/{id}", new { id });
});

app.Run();

// ---------- Command, validator y handler ----------
public sealed record CreateUsuarioCommand(string Nombre, string Email) : IRequest<Guid>;

public sealed class CreateUsuarioCommandValidator : AbstractValidator<CreateUsuarioCommand>
{
    public CreateUsuarioCommandValidator(UsuariosContext db)
    {
        RuleFor(x => x.Nombre)
            .NotEmpty().WithMessage("El nombre es obligatorio.")
            .MaximumLength(100);

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress().WithMessage("El correo no es válido.")
            .MustAsync(async (email, ct) => !await db.Usuarios.AnyAsync(u => u.Email == email, ct))
            .WithMessage("El correo ya está registrado.");
    }
}

public sealed class CreateUsuarioCommandHandler(UsuariosContext db) : IRequestHandler<CreateUsuarioCommand, Guid>
{
    public async Task<Guid> Handle(CreateUsuarioCommand request, CancellationToken cancellationToken)
    {
        var usuario = new Usuario { Nombre = request.Nombre.Trim(), Email = request.Email.Trim() };
        db.Usuarios.Add(usuario);
        await db.SaveChangesAsync(cancellationToken);
        return usuario.Id;
    }
}

// ---------- Pipeline ----------
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = new List<ValidationFailure>();

        foreach (var validator in validators)
        {
            var result = await validator.ValidateAsync(context, cancellationToken);
            failures.AddRange(result.Errors);
        }

        if (failures.Count > 0)
        {
            throw new ValidationException(failures);
        }

        return await next(cancellationToken);
    }
}

// ---------- HTTP ----------
public sealed class ValidationExceptionHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not ValidationException validation)
        {
            return false;
        }

        var errors = validation.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

        httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = new HttpValidationProblemDetails(errors),
            Exception = exception
        });
    }
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

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Usuario>(usuario =>
        {
            usuario.Property(u => u.Nombre).HasMaxLength(100);
            usuario.Property(u => u.Email).HasMaxLength(254);
            usuario.HasIndex(u => u.Email).IsUnique();   // la garantía real de unicidad
        });
    }
}
```

Peticiones (el Guid y el `traceId` cambian en cada ejecución):

```text
POST /usuarios  {"nombre":"Ana","email":"ana@example.com"}
→ 201  {"id":"{id}"}
       Location: /usuarios/{id}

POST /usuarios  {"nombre":"Ana Paula","email":"ana@example.com"}
→ 400  Content-Type: application/problem+json
       {"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1",
        "title":"One or more validation errors occurred.",
        "status":400,
        "errors":{"Email":["El correo ya está registrado."]},
        "traceId":"{traceId}"}

POST /usuarios  {"nombre":"","email":"no-es-un-correo"}
→ 400  ... "errors":{"Nombre":["El nombre es obligatorio."],"Email":["El correo no es válido."]} ...
```

Qué observar:

* En los dos `400`, el handler **no se ejecutó**: el behavior lanzó antes de llamar a `next`.
* El cuerpo es `HttpValidationProblemDetails`, el mismo formato que usa ASP.NET Core para errores de *model binding*. El cliente maneja un único formato de error.
* El endpoint es una Minimal API y el error igual se tradujo: `IExceptionHandler` no depende de MVC.
* El tercer caso junta errores de dos propiedades porque el behavior acumula todos los fallos antes de lanzar.
* Si dos peticiones con el mismo email pasan juntas el `AnyAsync`, el índice único hace fallar la segunda con `DbUpdateException` (un `500` en este ejemplo; el ejercicio 2 lo convierte en `400`).

-----

## Errores comunes

**1. `MustAsync(email => !context.X.AnyAsync(...))`.**
Qué pasa: `CS0023` por aplicar `!` a un `Task<bool>`, y la lambda no coincide con la firma de `MustAsync`.
Por qué: hay que esperar el `Task` antes de negarlo, y `MustAsync` pasa también el `CancellationToken`.
Arreglo: `MustAsync(async (email, ct) => !await ...AnyAsync(..., ct))`.

**2. Ejecutar validators con `Task.WhenAll`.**
Qué pasa: `InvalidOperationException` intermitente cuando dos validators consultan la base.
Por qué: comparten el mismo `DbContext` scoped, que no admite operaciones simultáneas.
Arreglo: `foreach` con `await` en secuencia.

**3. Confiar en el validator para la unicidad.**
Qué pasa: duplicados bajo concurrencia.
Por qué: condición de carrera entre la comprobación y el `INSERT`.
Arreglo: índice único en la base; el validator solo mejora el mensaje. Captura `DbUpdateException` donde corresponda.

**4. Traducir con un `IExceptionFilter` de MVC.**
Qué pasa: Minimal APIs y middleware devuelven `500` para el mismo error.
Por qué: los filtros de MVC solo envuelven acciones de controladores.
Arreglo: `IExceptionHandler` + `AddProblemDetails()` + `UseExceptionHandler()`.

**5. Restringir el behavior a `where TRequest : IRequest<TResponse>`.**
Qué pasa: los commands sin resultado (`IRequest`) no se validan, sin ningún aviso.
Por qué: en MediatR 12+, un `IRequest` sin genérico se procesa con `TResponse = Unit`, pero no implementa `IRequest<Unit>`; la restricción no se cumple y el behavior no se aplica a ese request.
Arreglo: `where TRequest : notnull`.

**6. Quitar toda validación del dominio "porque ya valida el pipeline".**
Qué pasa: un job en segundo plano crea entidades inválidas sin pasar por MediatR.
Por qué: el pipeline valida mensajes, no protege a la entidad.
Arreglo: invariantes en las entidades; el pipeline valida la forma de la entrada.

-----

## Según la versión de .NET

* **ASP.NET Core 2.1:** `[ApiController]` responde `400` con `ValidationProblemDetails` automáticamente.
* **ASP.NET Core 7:** `AddProblemDetails()` e `IProblemDetailsService`.
* **ASP.NET Core 8:** `IExceptionHandler`.
* **ASP.NET Core 10:** validación integrada para Minimal APIs con `AddValidation()`, basada en DataAnnotations.
* **MediatR 12:** `AddOpenBehavior`; **MediatR 12.5:** `next(cancellationToken)`; **MediatR 13:** licencia comercial.
* **FluentValidation 11:** se deja de mantener `FluentValidation.AspNetCore` (validación automática de MVC); la recomendación es validar manualmente o en un pipeline.

-----

## Cuándo sí y cuándo no

**Usa un validation behavior cuando:** ya usas MediatR y quieres que todos los commands se validen igual, sin repetir código en cada handler.

**Usa la validación integrada de ASP.NET Core cuando:** no usas MediatR, las reglas son simples (DataAnnotations) y basta con validar en el borde HTTP.

**No pongas en el validator:** reglas que dependen del estado del agregado (van en el dominio) ni la única protección de una regla de unicidad (va en la base).

-----

## Resumen en 5 líneas

1. Un pipeline behavior envuelve a todos los handlers; el de validación lanza antes de llamar a `next`.
2. `MustAsync` usa `(valor, ct)` y `!await`; los validators se ejecutan en secuencia.
3. `IExceptionHandler` traduce `ValidationException` a `HttpValidationProblemDetails` en toda la aplicación.
4. La unicidad la garantiza un índice único; el validator solo mejora el mensaje.
5. El pipeline valida la entrada; las invariantes siguen viviendo en el dominio.

-----

## Para profundizar

<details>
<summary>Convertir la violación del índice único en un 400</summary>

La idea: capturar `DbUpdateException`, confirmar que la causa fue el duplicado y relanzarla como `ValidationException`. La comprobación no puede ir en un filtro `when`, porque un filtro de excepción no admite `await` (CS7094); va dentro del `catch`, como en el ejercicio 2 de esta lección.

Detectar el código de error específico del motor (2601/2627 en SQL Server, 19 en SQLite) evita la consulta extra y es más preciso, pero ata el handler a un proveedor.

</details>

<details>
<summary>Otros behaviors útiles</summary>

* **Logging y métricas:** mide cuánto tarda cada request y registra su nombre.
* **Transacción:** abre una transacción antes de `next` y la confirma después, solo para commands.
* **Autorización:** comprueba permisos según atributos del request.

Cada behavior se aplica a **todos** los requests. Si uno solo debe actuar sobre commands, restringe el tipo con una interfaz marcadora (`where TRequest : ICommand`) y regístralo con esa restricción.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso FluentValidation para escribir las reglas de cada command y un `IPipelineBehavior` de MediatR que ejecuta los validators antes del handler. Si hay errores, lanza `ValidationException`, y un handler de excepciones la convierte en un `400` con ProblemDetails.

### Respuesta ampliada (semi-senior)

El behavior recibe todos los `IValidator<TRequest>` por DI, los ejecuta en secuencia (comparten el `DbContext`), acumula errores y lanza antes de llamar a `next`. Lo registro con `AddOpenBehavior` y restricción `notnull` para no dejar afuera los commands sin resultado. La traducción a HTTP la hago con `IExceptionHandler` y `HttpValidationProblemDetails`, que cubre controladores y Minimal APIs con el formato de RFC 9457. Separo tres niveles: forma de la entrada en el validator, invariantes en el dominio e integridad en la base; la unicidad la garantiza el índice, no el `AnyAsync`.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no `Task.WhenAll` para los validators?**
Porque comparten el `DbContext` scoped y no admite operaciones simultáneas.

**2. ¿Por qué `IExceptionHandler` y no un filtro de MVC?**
Porque cubre toda la aplicación, incluidas las Minimal APIs.

**3. ¿El validator garantiza que el email sea único?**
No; hay una condición de carrera. Lo garantiza el índice único.

**4. ¿En qué orden corren los behaviors?**
En el orden de registro: el primero registrado envuelve a los demás.

-----

## Práctica

**Ejercicio 1.** Escribe un `LoggingBehavior` que registre el nombre del request y cuánto tardó, y regístralo para que envuelva también a la validación.

<details>
<summary>Solución</summary>

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        long inicio = Stopwatch.GetTimestamp();
        try
        {
            return await next(cancellationToken);
        }
        finally
        {
            logger.LogInformation("{Request} tardó {Ms} ms",
                typeof(TRequest).Name, Stopwatch.GetElapsedTime(inicio).TotalMilliseconds);
        }
    }
}

builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));      // primero: el más externo
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

Al registrarlo primero, también mide (y registra) los requests que fallan en la validación, gracias al `finally`.

</details>

**Ejercicio 2.** Completa el handler del ejemplo para que un duplicado que se cuela por la condición de carrera responda `400` con el mismo mensaje, no `500`.

<details>
<summary>Solución</summary>

```csharp
public async Task<Guid> Handle(CreateUsuarioCommand request, CancellationToken cancellationToken)
{
    var usuario = new Usuario { Nombre = request.Nombre.Trim(), Email = request.Email.Trim() };
    db.Usuarios.Add(usuario);
    try
    {
        await db.SaveChangesAsync(cancellationToken);
    }
    catch (DbUpdateException)
    {
        db.Entry(usuario).State = EntityState.Detached;   // que un SaveChanges posterior no lo reintente
        if (await db.Usuarios.AnyAsync(u => u.Email == usuario.Email, cancellationToken))
        {
            throw new ValidationException([new ValidationFailure(nameof(request.Email), "El correo ya está registrado.")]);
        }
        throw;
    }
    return usuario.Id;
}
```

`ValidationExceptionHandler` ya sabe convertir esa excepción en `400`. Si el `DbUpdateException` tenía otra causa, `throw;` la deja subir intacta.

</details>

-----

## Siguiente lección

Terminaste el módulo. Vuelve al [índice](README.md).
