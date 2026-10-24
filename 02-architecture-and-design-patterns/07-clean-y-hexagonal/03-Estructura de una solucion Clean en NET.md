# Estructura de una solución Clean en .NET

## En una frase

Una solución Clean típica en .NET tiene cuatro proyectos (`Dominio`, `Aplicacion`, `Infraestructura` y `Api`) cuyas **referencias de proyecto** hacen cumplir la regla de dependencias en tiempo de compilación; cada capa expone un método `AddX()` para registrarse en DI y la API, como raíz de composición, solo los conecta.

-----

## Antes de empezar

Conviene que ya sepas:

* La regla de dependencias y los puertos: [Capas y la regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md) y [Puertos y adaptadores](02-Puertos%20y%20adaptadores.md).
* Mapear un dominio rico con Fluent API (campos de respaldo, constructores privados): [Modelado con convenciones, atributos y Fluent API](../../04-data-access-and-transactions/01-entity-framework-core/03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md).
* `IExceptionHandler` y ProblemDetails: [Errores con ProblemDetails](../../03-aspnet-core-apis/01-diseno-de-apis-rest/02-Errores%20con%20ProblemDetails.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Referencia de proyecto:** dependencia declarada en el `.csproj` (`<ProjectReference>`); el compilador impide usar tipos de proyectos no referenciados.
* **Método de extensión de registro:** método `AddAplicacion()`/`AddInfraestructura()` con el que cada capa registra sus propios servicios.
* **`internal`:** modificador que limita un tipo a su propio proyecto; sirve para ocultar detalles de infraestructura.
* **DTO de aplicación:** tipo que devuelve un caso de uso; la presentación no ve entidades del dominio.

-----

## El problema

Un equipo decide "hacer Clean" y crea cuatro proyectos. Seis meses después:

```text
Tienda.Api ──► Tienda.Aplicacion ──► Tienda.Dominio
    │                 │
    │                 └──► Tienda.Infraestructura   ← "para usar el DbContext en un caso de uso"
    └──► Tienda.Infraestructura
Tienda.Dominio ──► Microsoft.EntityFrameworkCore      ← "para poner [Index] en Pedido"
```

Y en los endpoints:

```csharp
app.MapGet("/pedidos/{id}", async (Guid id, TiendaDbContext db) =>
    await db.Pedidos.Include(p => p.Lineas).SingleAsync(p => p.Id == id));   // devuelve la entidad
```

Lo que pasa:

1. **La aplicación referencia a la infraestructura.** La flecha volvió a apuntar hacia afuera; los casos de uso usan `TiendaDbContext` y no se pueden probar sin base.
2. **El dominio referencia EF Core** por un atributo. Ya no es independiente.
3. **La API se saltea la aplicación** y devuelve entidades: ciclos de serialización, fuga del modelo interno y reglas que nunca se ejecutan.
4. **`Program.cs` tiene 200 líneas** registrando cada repositorio, servicio y caso de uso de todas las capas.

Cuatro proyectos con las referencias equivocadas son N-capas con más carpetas.

Las consecuencias:

* la estructura promete una arquitectura que el código no cumple;
* el costo de Clean (más proyectos, más mapeos) sin ninguno de sus beneficios.

-----

## Cómo funciona

### 1. Proyectos y referencias

```text
dotnet new sln -n Tienda
dotnet new classlib -o src/Tienda.Dominio
dotnet new classlib -o src/Tienda.Aplicacion
dotnet new classlib -o src/Tienda.Infraestructura
dotnet new web      -o src/Tienda.Api
dotnet sln add src/Tienda.Dominio src/Tienda.Aplicacion src/Tienda.Infraestructura src/Tienda.Api

dotnet add src/Tienda.Aplicacion      reference src/Tienda.Dominio
dotnet add src/Tienda.Infraestructura reference src/Tienda.Aplicacion
dotnet add src/Tienda.Api             reference src/Tienda.Aplicacion src/Tienda.Infraestructura
```

```text
          Tienda.Api  (raíz de composición, adaptadores HTTP)
           │       │
           ▼       ▼
 Tienda.Aplicacion ◄── Tienda.Infraestructura (EF Core, SQLite, adaptadores de salida)
           │
           ▼
      Tienda.Dominio  (sin paquetes NuGet)
```

| Proyecto | Paquetes NuGet | Referencia a |
| --- | --- | --- |
| `Tienda.Dominio` | Ninguno | Nadie |
| `Tienda.Aplicacion` | `Microsoft.Extensions.DependencyInjection.Abstractions` | Dominio |
| `Tienda.Infraestructura` | `Microsoft.EntityFrameworkCore.Sqlite` | Aplicación (y Dominio por transitividad) |
| `Tienda.Api` | — (framework web) | Aplicación, Infraestructura |

Con esto, escribir `using Microsoft.EntityFrameworkCore;` en el dominio **no compila**: el compilador hace cumplir la regla.

La API referencia a Infraestructura **solo** para llamar a `AddInfraestructura()` en la composición. Para evitar que los endpoints usen tipos de infraestructura, esos tipos son `internal`.

### 2. Qué va en cada proyecto

| Proyecto | Carpetas típicas | Ejemplos |
| --- | --- | --- |
| Dominio | `Pedidos/`, `Comun/` | `Pedido`, `LineaPedido`, `EstadoPedido`, `DomainException` |
| Aplicación | `Puertos/`, `Pedidos/` (casos de uso), `DependencyInjection.cs` | `IPedidoRepository`, `CrearPedido`, `ConfirmarPedido`, `PedidoDto` |
| Infraestructura | `Persistencia/`, `Configuraciones/`, `DependencyInjection.cs` | `TiendaDbContext`, `PedidoConfiguration`, `PedidoRepositoryEf` |
| Api | `Endpoints/`, `Errores/`, `Program.cs` | Endpoints, `DomainExceptionHandler`, requests HTTP |

Organizar **por funcionalidad** dentro de cada proyecto (`Pedidos/`, `Clientes/`) escala mejor que por tipo técnico (`Repositories/`, `Services/`, `DTOs/`).

### 3. Cada capa se registra sola

```csharp
// Tienda.Aplicacion/DependencyInjection.cs
public static IServiceCollection AddAplicacion(this IServiceCollection services)
{
    services.AddScoped<CrearPedido>();
    services.AddScoped<ConfirmarPedido>();
    return services;
}

// Tienda.Infraestructura/DependencyInjection.cs
public static IServiceCollection AddInfraestructura(this IServiceCollection services, string cadenaConexion)
{
    services.AddDbContext<TiendaDbContext>(o => o.UseSqlite(cadenaConexion));
    services.AddScoped<IPedidoRepository, PedidoRepositoryEf>();
    return services;
}

// Tienda.Api/Program.cs
builder.Services.AddAplicacion();
builder.Services.AddInfraestructura(cadena);
```

`Program.cs` queda corto y cada capa es dueña de sus registros. Como `TiendaDbContext` y `PedidoRepositoryEf` son `internal`, solo `AddInfraestructura` (que está en su mismo proyecto) puede registrarlos.

### 4. Un dominio rico mapeado desde afuera

El dominio tiene constructores privados, *setters* privados y una lista privada. EF Core los maneja sin que el dominio lo sepa:

```csharp
internal sealed class PedidoConfiguration : IEntityTypeConfiguration<Pedido>
{
    public void Configure(EntityTypeBuilder<Pedido> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Estado).HasConversion<string>().HasMaxLength(20);
        builder.HasMany(p => p.Lineas).WithOne().HasForeignKey("PedidoId").IsRequired();      // FK como propiedad sombra
        builder.Navigation(p => p.Lineas).UsePropertyAccessMode(PropertyAccessMode.Field);   // usa _lineas
    }
}
```

* `private Pedido() { }` le permite a EF Core materializar la entidad sin pasar por `Pedido.Crear`.
* La FK `PedidoId` es una **propiedad sombra**: existe en la tabla y en EF Core, no en el dominio.
* `Total` y `Subtotal` son propiedades calculadas sin `set`: no se mapean.

### 5. Los casos de uso devuelven DTOs, no entidades

```csharp
public sealed record PedidoDto(Guid Id, int ClienteId, string Estado, string Total, IReadOnlyList<LineaDto> Lineas);
```

La API nunca recibe un `Pedido`: no puede llamar a sus métodos saltándose el caso de uso, ni serializarlo por accidente. Los DTOs de la aplicación son el contrato entre el núcleo y cualquier adaptador de entrada; los requests HTTP son de la API.

### 6. Errores del dominio → HTTP en la API

El dominio lanza `DomainException`; la aplicación no sabe de HTTP; la API traduce en un solo lugar:

```csharp
internal sealed class DomainExceptionHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext context, Exception exception, CancellationToken ct)
    {
        (int status, string titulo) = exception switch
        {
            DomainException => (StatusCodes.Status422UnprocessableEntity, "Regla de negocio incumplida"),
            ArgumentException => (StatusCodes.Status400BadRequest, "Datos inválidos"),
            _ => (0, "")
        };
        if (status == 0) return false;

        context.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            ProblemDetails = { Status = status, Title = titulo, Detail = exception.Message },
            Exception = exception
        });
    }
}
```

### 7. Una alternativa que vas a ver: `IApplicationDbContext`

Plantillas populares de Clean Architecture para .NET exponen en la aplicación una interfaz con los `DbSet` (`IApplicationDbContext`) y escriben las consultas de EF Core en los casos de uso:

| | Repositorio por agregado | `IApplicationDbContext` |
| --- | --- | --- |
| La aplicación depende de EF Core | No | Sí (`DbSet<T>`, `Include`, `ToListAsync`) |
| Consultas de lectura | En el repositorio o en un puerto de consultas | En el caso de uso, con LINQ completo |
| Probar casos de uso | Fakes simples del repositorio | Base real o SQLite (mockear `DbSet` miente) |
| Código | Más | Menos |

Es una concesión pragmática y válida si el equipo la elige con conciencia: la aplicación pasa a depender de EF Core, que es una abstracción bastante estable. Lo que no es válido es elegirla creyendo que sigue siendo "independiente de la infraestructura".

-----

## Ejemplo completo

Solución con los cuatro proyectos del punto 1. Paquetes: `Microsoft.Extensions.DependencyInjection.Abstractions` 10.0.0 en `Tienda.Aplicacion` y `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0 en `Tienda.Infraestructura`.

`src/Tienda.Dominio/Pedidos.cs`:

```csharp
namespace Tienda.Dominio;

public sealed class DomainException(string mensaje) : Exception(mensaje);

public enum EstadoPedido { Borrador, Confirmado }

public sealed class Pedido
{
    private readonly List<LineaPedido> _lineas = [];

    private Pedido() { }   // para EF Core

    public Guid Id { get; private set; }
    public int ClienteId { get; private set; }
    public EstadoPedido Estado { get; private set; }
    public IReadOnlyCollection<LineaPedido> Lineas => _lineas.AsReadOnly();
    public decimal Total => _lineas.Sum(l => l.Subtotal);

    public static Pedido Crear(int clienteId)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(clienteId);
        return new Pedido { Id = Guid.NewGuid(), ClienteId = clienteId, Estado = EstadoPedido.Borrador };
    }

    public void AgregarLinea(string producto, decimal precioUnitario, int cantidad)
    {
        if (Estado != EstadoPedido.Borrador) throw new DomainException("El pedido ya no admite cambios.");
        ArgumentException.ThrowIfNullOrWhiteSpace(producto);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(precioUnitario);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(cantidad);
        _lineas.Add(new LineaPedido(producto.Trim(), precioUnitario, cantidad));
    }

    public void Confirmar()
    {
        if (Estado != EstadoPedido.Borrador) throw new DomainException("El pedido ya está confirmado.");
        if (_lineas.Count == 0) throw new DomainException("No se puede confirmar un pedido vacío.");
        Estado = EstadoPedido.Confirmado;
    }
}

public sealed class LineaPedido
{
    private LineaPedido() { }   // para EF Core

    internal LineaPedido(string producto, decimal precioUnitario, int cantidad) =>
        (Producto, PrecioUnitario, Cantidad) = (producto, precioUnitario, cantidad);

    public int Id { get; private set; }
    public string Producto { get; private set; } = "";
    public decimal PrecioUnitario { get; private set; }
    public int Cantidad { get; private set; }
    public decimal Subtotal => PrecioUnitario * Cantidad;
}
```

`src/Tienda.Aplicacion/Pedidos.cs`:

```csharp
using System.Globalization;
using Tienda.Dominio;

namespace Tienda.Aplicacion;

// Puerto de salida: lo que la aplicación necesita para persistir el agregado
public interface IPedidoRepository
{
    Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct);
    void Agregar(Pedido pedido);
    Task GuardarCambiosAsync(CancellationToken ct);
}

// DTOs de la aplicación: el dinero viaja como texto con dos decimales
public sealed record LineaDto(string Producto, string PrecioUnitario, int Cantidad);
public sealed record PedidoDto(Guid Id, int ClienteId, string Estado, string Total, IReadOnlyList<LineaDto> Lineas);

public sealed class CrearPedido(IPedidoRepository repositorio)
{
    public async Task<Guid> EjecutarAsync(int clienteId, CancellationToken ct)
    {
        var pedido = Pedido.Crear(clienteId);
        repositorio.Agregar(pedido);
        await repositorio.GuardarCambiosAsync(ct);
        return pedido.Id;
    }
}

public sealed class AgregarLinea(IPedidoRepository repositorio)
{
    public async Task<bool> EjecutarAsync(Guid pedidoId, string producto, decimal precio, int cantidad, CancellationToken ct)
    {
        Pedido? pedido = await repositorio.ObtenerAsync(pedidoId, ct);
        if (pedido is null) return false;

        pedido.AgregarLinea(producto, precio, cantidad);   // las reglas las decide el dominio
        await repositorio.GuardarCambiosAsync(ct);
        return true;
    }
}

public sealed class ConfirmarPedido(IPedidoRepository repositorio)
{
    public async Task<bool> EjecutarAsync(Guid pedidoId, CancellationToken ct)
    {
        Pedido? pedido = await repositorio.ObtenerAsync(pedidoId, ct);
        if (pedido is null) return false;

        pedido.Confirmar();
        await repositorio.GuardarCambiosAsync(ct);
        return true;
    }
}

public sealed class ObtenerPedido(IPedidoRepository repositorio)
{
    public async Task<PedidoDto?> EjecutarAsync(Guid pedidoId, CancellationToken ct)
    {
        Pedido? pedido = await repositorio.ObtenerAsync(pedidoId, ct);
        if (pedido is null) return null;

        return new PedidoDto(
            pedido.Id,
            pedido.ClienteId,
            pedido.Estado.ToString(),
            Formatear(pedido.Total),
            pedido.Lineas.OrderBy(l => l.Id).Select(l => new LineaDto(l.Producto, Formatear(l.PrecioUnitario), l.Cantidad)).ToList());
    }

    private static string Formatear(decimal monto) => monto.ToString("F2", CultureInfo.InvariantCulture);
}
```

`src/Tienda.Aplicacion/DependencyInjection.cs`:

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace Tienda.Aplicacion;

public static class DependencyInjection
{
    public static IServiceCollection AddAplicacion(this IServiceCollection services)
    {
        services.AddScoped<CrearPedido>();
        services.AddScoped<AgregarLinea>();
        services.AddScoped<ConfirmarPedido>();
        services.AddScoped<ObtenerPedido>();
        return services;
    }
}
```

`src/Tienda.Infraestructura/Persistencia.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.Extensions.DependencyInjection;
using Tienda.Aplicacion;
using Tienda.Dominio;

namespace Tienda.Infraestructura;

internal sealed class TiendaDbContext(DbContextOptions<TiendaDbContext> options) : DbContext(options)
{
    public DbSet<Pedido> Pedidos => Set<Pedido>();

    protected override void OnModelCreating(ModelBuilder modelBuilder) =>
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(TiendaDbContext).Assembly);
}

internal sealed class PedidoConfiguration : IEntityTypeConfiguration<Pedido>
{
    public void Configure(EntityTypeBuilder<Pedido> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Estado).HasConversion<string>().HasMaxLength(20);
        builder.HasMany(p => p.Lineas).WithOne().HasForeignKey("PedidoId").IsRequired();
        builder.Navigation(p => p.Lineas).UsePropertyAccessMode(PropertyAccessMode.Field);
    }
}

internal sealed class LineaPedidoConfiguration : IEntityTypeConfiguration<LineaPedido>
{
    public void Configure(EntityTypeBuilder<LineaPedido> builder)
    {
        builder.HasKey(l => l.Id);
        builder.Property(l => l.Producto).HasMaxLength(200);
    }
}

// Adaptador de salida: implementa el puerto de la aplicación con EF Core
internal sealed class PedidoRepositoryEf(TiendaDbContext db) : IPedidoRepository
{
    public Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct) =>
        db.Pedidos.Include(p => p.Lineas).SingleOrDefaultAsync(p => p.Id == id, ct);

    public void Agregar(Pedido pedido) => db.Pedidos.Add(pedido);

    public Task GuardarCambiosAsync(CancellationToken ct) => db.SaveChangesAsync(ct);
}

public static class DependencyInjection
{
    public static IServiceCollection AddInfraestructura(this IServiceCollection services, string cadenaConexion)
    {
        services.AddDbContext<TiendaDbContext>(o => o.UseSqlite(cadenaConexion));
        services.AddScoped<IPedidoRepository, PedidoRepositoryEf>();
        return services;
    }

    // Solo para el ejemplo: en una aplicación real se aplican migraciones
    public static async Task InicializarBaseDeDatosAsync(this IServiceProvider services)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<TiendaDbContext>();
        await db.Database.EnsureDeletedAsync();
        await db.Database.EnsureCreatedAsync();
    }
}
```

`src/Tienda.Api/Program.cs`:

```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Http.HttpResults;
using Tienda.Aplicacion;
using Tienda.Dominio;
using Tienda.Infraestructura;

var builder = WebApplication.CreateBuilder(args);

// Raíz de composición: cada capa registra lo suyo
builder.Services.AddAplicacion();
builder.Services.AddInfraestructura(builder.Configuration.GetConnectionString("Tienda") ?? "Data Source=tienda.db");
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();

var app = builder.Build();
await app.Services.InicializarBaseDeDatosAsync();
app.UseExceptionHandler();

// Adaptadores de entrada: traducen HTTP ↔ casos de uso
var pedidos = app.MapGroup("/pedidos");

pedidos.MapPost("/", async (CrearPedidoRequest request, CrearPedido casoDeUso, CancellationToken ct) =>
{
    Guid id = await casoDeUso.EjecutarAsync(request.ClienteId, ct);
    return TypedResults.Created($"/pedidos/{id}", new { id });
});

pedidos.MapPost("/{id:guid}/lineas", async Task<Results<NoContent, NotFound>> (
    Guid id, AgregarLineaRequest request, AgregarLinea casoDeUso, CancellationToken ct) =>
        await casoDeUso.EjecutarAsync(id, request.Producto, request.Precio, request.Cantidad, ct)
            ? TypedResults.NoContent()
            : TypedResults.NotFound());

pedidos.MapPost("/{id:guid}/confirmar", async Task<Results<NoContent, NotFound>> (
    Guid id, ConfirmarPedido casoDeUso, CancellationToken ct) =>
        await casoDeUso.EjecutarAsync(id, ct) ? TypedResults.NoContent() : TypedResults.NotFound());

pedidos.MapGet("/{id:guid}", async Task<Results<Ok<PedidoDto>, NotFound>> (
    Guid id, ObtenerPedido casoDeUso, CancellationToken ct) =>
        await casoDeUso.EjecutarAsync(id, ct) is { } dto ? TypedResults.Ok(dto) : TypedResults.NotFound());

app.Run();

// Requests HTTP: son de la API, no de la aplicación
public sealed record CrearPedidoRequest(int ClienteId);
public sealed record AgregarLineaRequest(string Producto, decimal Precio, int Cantidad);

internal sealed class DomainExceptionHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext context, Exception exception, CancellationToken ct)
    {
        (int status, string titulo) = exception switch
        {
            DomainException => (StatusCodes.Status422UnprocessableEntity, "Regla de negocio incumplida"),
            ArgumentException => (StatusCodes.Status400BadRequest, "Datos inválidos"),
            _ => (0, "")
        };
        if (status == 0) return false;

        context.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            ProblemDetails = { Status = status, Title = titulo, Detail = exception.Message },
            Exception = exception
        });
    }
}

public partial class Program { }
```

Peticiones (el Guid cambia en cada ejecución; aquí `{id}`):

```text
POST /pedidos  {"clienteId":7}
→ 201 {"id":"{id}"}   Location: /pedidos/{id}

POST /pedidos/{id}/confirmar
→ 422 {"title":"Regla de negocio incumplida","status":422,"detail":"No se puede confirmar un pedido vacío.", ...}

POST /pedidos/{id}/lineas  {"producto":"Teclado","precio":150,"cantidad":1}   → 204
POST /pedidos/{id}/lineas  {"producto":"Mouse","precio":60,"cantidad":2}      → 204
POST /pedidos/{id}/lineas  {"producto":"Cable","precio":10,"cantidad":0}      → 400 "Datos inválidos"
POST /pedidos/{id}/confirmar                                                  → 204

GET /pedidos/{id}
→ 200 {"id":"{id}","clienteId":7,"estado":"Confirmado","total":"270.00",
       "lineas":[{"producto":"Teclado","precioUnitario":"150.00","cantidad":1},
                 {"producto":"Mouse","precioUnitario":"60.00","cantidad":2}]}

POST /pedidos/{id}/lineas  {"producto":"Cable","precio":10,"cantidad":1}      → 422 "El pedido ya no admite cambios."
POST /pedidos/00000000-0000-0000-0000-000000000001/confirmar                  → 404
```

Qué observar:

* `Tienda.Dominio` no tiene paquetes NuGet ni referencias: las reglas (pedido vacío, pedido confirmado, cantidad positiva) se ejercen igual desde HTTP que desde una prueba unitaria.
* Los endpoints solo conocen casos de uso y DTOs. No pueden usar `TiendaDbContext`: es `internal` de Infraestructura.
* El `422` y el `400` salen de un único `DomainExceptionHandler`. Ni el dominio ni la aplicación saben qué es un código HTTP.
* EF Core guardó y recuperó un agregado con constructor privado, *setters* privados y lista privada, sin un solo atributo en el dominio.
* El dinero viaja como texto con dos decimales (`"270.00"`): es una decisión del DTO, que evita depender de cómo serializa cada proveedor un `decimal` y de la precisión de los números en JavaScript.
* `Program.cs` tiene cuatro líneas de registro; si mañana Infraestructura usa PostgreSQL, cambia `AddInfraestructura` y nada más.

-----

## Errores comunes

**1. La aplicación referencia a Infraestructura.**
Qué pasa: los casos de uso usan el `DbContext` o clases concretas; no se prueban sin base.
Por qué: alguien agregó la referencia "para una cosa rápida".
Arreglo: la aplicación define el puerto; Infraestructura referencia a Aplicación, nunca al revés.

**2. Paquetes de infraestructura en el dominio.**
Qué pasa: `Microsoft.EntityFrameworkCore` en `Tienda.Dominio` por un `[Index]` o un `[Owned]`.
Por qué: los atributos parecen inofensivos.
Arreglo: todo el mapeo en `IEntityTypeConfiguration` dentro de Infraestructura.

**3. Tipos de infraestructura públicos.**
Qué pasa: los endpoints inyectan `TiendaDbContext` directamente y se saltean los casos de uso.
Por qué: la API referencia a Infraestructura y todo es `public`.
Arreglo: `internal` para el `DbContext`, los repositorios y las configuraciones; solo el método `AddInfraestructura` es público.

**4. Casos de uso que devuelven entidades.**
Qué pasa: la API serializa el agregado (ciclos, campos internos) o llama a sus métodos sin pasar por la aplicación.
Por qué: devolver la entidad "ahorra un DTO".
Arreglo: DTOs de aplicación para todo lo que sale del núcleo.

**5. Registrar todo en `Program.cs`.**
Qué pasa: un archivo enorme que conoce cada clase de cada capa, y tipos `internal` que no se pueden registrar desde afuera.
Por qué: falta un punto de registro por capa.
Arreglo: `AddAplicacion()` y `AddInfraestructura()` en sus proyectos.

**6. Proyectos separados sin beneficio.**
Qué pasa: `Tienda.Contratos`, `Tienda.Comun`, `Tienda.Utilidades`, `Tienda.Interfaces`... diez proyectos para tres casos de uso.
Por qué: más proyectos se confunde con más arquitectura.
Arreglo: los cuatro de esta lección alcanzan para la mayoría de los sistemas; separa más solo por una razón concreta (despliegue independiente, equipos distintos).

-----

## Según la versión de .NET

* **.NET Core 2.0:** `netstandard2.0` y referencias de proyecto con el SDK nuevo, más simples de mantener.
* **EF Core 1.1–2.0:** campos de respaldo y propiedades sombra, base del mapeo sin contaminar el dominio.
* **ASP.NET Core 6:** Minimal APIs, adaptadores de entrada muy livianos.
* **ASP.NET Core 8:** `IExceptionHandler`, un único lugar para traducir errores del dominio a HTTP.
* **.NET 10:** soluciones `.slnx` por defecto en `dotnet new sln`.

-----

## Cuándo sí y cuándo no

**Usa esta estructura cuando:** el dominio tiene reglas propias, el sistema crece con varios casos de uso y equipos, y quieres que el compilador proteja la arquitectura.

**Usa un solo proyecto con carpetas cuando:** el sistema es chico o el equipo es de una persona, y aun así quieres respetar la dirección de dependencias (verificándola con pruebas de arquitectura).

**No separes en proyectos cuando:** casi todo es CRUD: el costo de mapear entre capas supera al beneficio ([lección 4](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md)).

-----

## Resumen en 5 líneas

1. Cuatro proyectos: Dominio (sin dependencias), Aplicación → Dominio, Infraestructura → Aplicación, Api → ambos.
2. Las referencias de proyecto hacen que violar la regla no compile.
3. Cada capa se registra con su `AddX()`; los detalles de infraestructura son `internal`.
4. El dominio rico se mapea desde Infraestructura con Fluent API: constructores privados, campos de respaldo y propiedades sombra.
5. Los casos de uso devuelven DTOs; la API traduce errores del dominio a HTTP en un solo lugar.

-----

## Para profundizar

<details>
<summary>Un puerto de consultas para las lecturas</summary>

`ObtenerPedido` carga el agregado completo para armar un DTO. Para pantallas con listados, filtros o datos de varios agregados, conviene un puerto específico de lectura (`IConsultasPedidos.ListarPendientesAsync()`) implementado en Infraestructura con una proyección de EF Core o con Dapper. Es CQRS aplicado a la arquitectura: el repositorio protege la escritura del agregado y las consultas van por un camino optimizado ([CQRS y mediator](../06-cqrs-y-mediator/02-CQRS%20y%20mediator%20con%20MediatR.md), [Dapper](../../04-data-access-and-transactions/02-ado-net-y-dapper/README.md)).

</details>

<details>
<summary>Con MediatR: commands en Aplicación, endpoints en la API</summary>

Si usas MediatR, los casos de uso se vuelven `IRequest` + `IRequestHandler` dentro de `Tienda.Aplicacion`, `AddAplicacion()` llama a `AddMediatR(...)` con el ensamblado de la aplicación, y los endpoints hacen `sender.Send(...)`. Los *pipeline behaviors* (validación, logs, transacciones) viven también en Aplicación. La estructura de proyectos no cambia.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Separo la solución en Dominio, Aplicación, Infraestructura y Api. El dominio no depende de nada, la aplicación depende del dominio, la infraestructura implementa las interfaces de la aplicación con EF Core, y la API conecta todo y expone los endpoints.

### Respuesta ampliada (semi-senior)

Uso referencias de proyecto para que la regla de dependencias la verifique el compilador: el dominio sin paquetes, la aplicación con sus puertos y casos de uso, la infraestructura con EF Core y tipos `internal`, y la API como raíz de composición con `AddAplicacion()` y `AddInfraestructura()`. Mapeo el dominio rico con Fluent API (constructores privados, campos de respaldo, propiedades sombra), los casos de uso devuelven DTOs y traduzco las excepciones del dominio a ProblemDetails con un `IExceptionHandler`. Conozco la variante `IApplicationDbContext`: menos código, pero la aplicación depende de EF Core, y la elijo solo si el equipo lo acepta conscientemente.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué la API referencia a Infraestructura si no debería usarla?**
Para registrarla en la composición; los tipos `internal` impiden que los endpoints la usen.

**2. ¿Cómo mapea EF Core una entidad con constructor privado?**
EF Core puede usar constructores privados y campos de respaldo; el dominio no necesita cambiar.

**3. ¿Dónde se traducen las excepciones del dominio a códigos HTTP?**
En la API, con un `IExceptionHandler`.

**4. ¿Repositorio o `IApplicationDbContext`?**
Repositorio si quieres la aplicación libre de EF Core; `IApplicationDbContext` si aceptas esa dependencia a cambio de menos código.

-----

## Práctica

**Ejercicio 1.** Agrega el caso de uso "quitar línea" de punta a punta: método en el dominio, caso de uso, registro y endpoint `DELETE /pedidos/{id}/lineas/{producto}`. Indica qué proyecto tocas en cada paso.

<details>
<summary>Solución</summary>

**Dominio** (`Pedido`):

```csharp
public void QuitarLinea(string producto)
{
    if (Estado != EstadoPedido.Borrador) throw new DomainException("El pedido ya no admite cambios.");
    int quitadas = _lineas.RemoveAll(l => string.Equals(l.Producto, producto, StringComparison.OrdinalIgnoreCase));
    if (quitadas == 0) throw new DomainException($"El pedido no tiene el producto '{producto}'.");
}
```

**Aplicación**:

```csharp
public sealed class QuitarLinea(IPedidoRepository repositorio)
{
    public async Task<bool> EjecutarAsync(Guid pedidoId, string producto, CancellationToken ct)
    {
        Pedido? pedido = await repositorio.ObtenerAsync(pedidoId, ct);
        if (pedido is null) return false;
        pedido.QuitarLinea(producto);
        await repositorio.GuardarCambiosAsync(ct);
        return true;
    }
}
// en AddAplicacion: services.AddScoped<QuitarLinea>();
```

**Api**:

```csharp
pedidos.MapDelete("/{id:guid}/lineas/{producto}", async Task<Results<NoContent, NotFound>> (
    Guid id, string producto, QuitarLinea casoDeUso, CancellationToken ct) =>
        await casoDeUso.EjecutarAsync(id, producto, ct) ? TypedResults.NoContent() : TypedResults.NotFound());
```

**Infraestructura no cambia**: al quitar la línea de `_lineas`, EF Core detecta que el dependiente quedó huérfano y, como la relación es obligatoria, lo borra en `SaveChangesAsync`.

</details>

**Ejercicio 2.** Un compañero agrega `services.AddScoped<TiendaDbContext>()` en `Program.cs` para usarlo en un endpoint de reportes. ¿Qué error obtiene y qué le propones?

<details>
<summary>Solución</summary>

No compila: `CS0122: 'TiendaDbContext' is inaccessible due to its protection level`, porque es `internal` de Infraestructura. Es la protección funcionando. La propuesta es un puerto de consultas en la aplicación (`IReportesPedidos.VentasPorDiaAsync()`), implementado en Infraestructura con una proyección de EF Core o Dapper, y registrado en `AddInfraestructura`. El endpoint depende del puerto, y la consulta queda optimizada y en su capa.

</details>

-----

## Siguiente lección

[Pruebas de arquitectura y cuándo no usar Clean](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md)
