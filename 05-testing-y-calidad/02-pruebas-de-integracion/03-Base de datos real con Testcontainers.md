# Base de datos real con Testcontainers

## En una frase

Testcontainers arranca desde el código de prueba un contenedor Docker con el **mismo motor** que usas en producción (PostgreSQL, SQL Server), le pasa su cadena de conexión a la API y lo destruye al terminar; así las pruebas ejercen índices, restricciones, transacciones y traducción de SQL reales, que ningún proveedor en memoria reproduce.

-----

## Antes de empezar

Conviene que ya sepas:

* Fábricas propias y `UseSetting`: [WebApplicationFactory y pruebas de API en memoria](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md).
* EF Core, índices únicos y `DbUpdateException`: [CRUD en APIs con EF Core](../../04-data-access-and-transactions/01-entity-framework-core/06-CRUD%20en%20APIs%20con%20EF%20Core.md).
* Imágenes y contenedores: [Contenedores Docker para APIs .NET](../../02-architecture-and-design-patterns/05-cloud-native-y-contenedores/01-Contenedores%20Docker%20para%20APIs%20NET.md).

Necesitas **Docker** (Docker Desktop, Rancher Desktop o Podman compatible) en ejecución para correr el ejemplo.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Testcontainers:** biblioteca que crea, configura y elimina contenedores Docker desde el código de prueba.
* **`IAsyncLifetime`:** interfaz de xUnit para inicializar y liberar recursos de forma asíncrona antes y después de las pruebas.
* **Proveedor InMemory:** proveedor de EF Core que guarda datos en colecciones en memoria; no es una base relacional.
* **Paridad con producción:** que el entorno de prueba se comporte como el real en lo que importa.

-----

## El problema

Las pruebas de integración de la API usan el proveedor InMemory de EF Core "porque es rápido y no necesita nada":

```csharp
services.AddDbContext<TareasContext>(o => o.UseInMemoryDatabase("pruebas"));
```

Todas pasan. En producción, con PostgreSQL:

1. **Duplicados.** La tabla tiene un índice único sobre `Titulo`. Con InMemory, `POST` con un título repetido devuelve `201`: el índice **no se aplica**. En producción lanza `DbUpdateException` y la API responde `500`, porque nadie escribió el manejo del duplicado (ninguna prueba lo pedía).
2. **Consultas que no se traducen.** Un filtro con un método propio de C# funciona en InMemory (que evalúa todo en memoria) y lanza `InvalidOperationException: The LINQ expression ... could not be translated` en PostgreSQL.
3. **Mayúsculas.** Una búsqueda por `Titulo == "pagar la luz"` encuentra "Pagar la luz" en una base configurada sin distinguir mayúsculas, y no la encuentra en PostgreSQL, que sí distingue. El comportamiento cambia según el motor.
4. **Transacciones.** InMemory ignora las transacciones: un rollback no deshace nada.

El equipo prueba con SQLite en memoria. Mejora (es relacional y aplica índices), pero tiene sus propios tipos, funciones y reglas de mayúsculas: sigue sin ser PostgreSQL.

Las consecuencias:

* pruebas verdes sobre un comportamiento que producción no tiene;
* errores de restricciones, traducción y colación descubiertos por los usuarios.

-----

## Cómo funciona

### 1. Las opciones, sin rodeos

| Opción | Restricciones (únicos, FK) | Traducción SQL real | Transacciones | Paridad con PostgreSQL | Requiere |
| --- | --- | --- | --- | --- | --- |
| EF Core InMemory | ❌ | ❌ (evalúa en memoria) | ❌ | Ninguna | Nada |
| SQLite en memoria | ✔ | ✔, pero de SQLite | ✔ | Parcial | Nada |
| Base compartida de desarrollo | ✔ | ✔ | ✔ | Total | Una base que todos pisan |
| **Testcontainers** | ✔ | ✔ | ✔ | **Total** | Docker |

Microsoft desaconseja el proveedor InMemory para probar código que accede a datos. Testcontainers da paridad total con una base **propia y desechable** por ejecución.

### 2. Qué hace Testcontainers

```text
InitializeAsync()                                      DisposeAsync()
   │                                                        │
   ├─► docker pull postgres:17-alpine (solo la 1.ª vez)     ├─► detiene y borra el contenedor
   ├─► docker run con puerto aleatorio y credenciales       │
   ├─► espera a que PostgreSQL acepte conexiones            │
   └─► GetConnectionString() → "Host=localhost;Port=54871;..."
```

* El **puerto es aleatorio**: varias ejecuciones (o varios proyectos) no chocan.
* Espera a que el motor esté listo antes de devolver el control: no hay que poner `Task.Delay`.
* Un contenedor auxiliar (Ryuk) borra los contenedores huérfanos si el proceso de pruebas muere a mitad de camino.

### 3. La API tiene que leer la cadena de conexión de la configuración

```csharp
builder.Services.AddDbContext<TareasContext>((sp, opciones) =>
    opciones.UseNpgsql(sp.GetRequiredService<IConfiguration>().GetConnectionString("Tareas")));
```

Leer la cadena **dentro** del lambda (y no en una variable al inicio de `Program.cs`) hace que se resuelva cuando se crea el contexto, después de que la fábrica de pruebas agregó su configuración. Así la prueba solo tiene que sobrescribir un valor:

```csharp
builder.UseSetting("ConnectionStrings:Tareas", _postgres.GetConnectionString());
```

Es más simple y más seguro que quitar el registro del `DbContext` y volver a agregarlo (ver el error común 2).

### 4. La fábrica con el contenedor

```csharp
public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder) =>
        builder.UseSetting("ConnectionStrings:Tareas", _postgres.GetConnectionString());

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        using var scope = Services.CreateScope();   // construye la app (ya con la cadena del contenedor)
        await scope.ServiceProvider.GetRequiredService<TareasContext>().Database.EnsureCreatedAsync();
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await base.DisposeAsync();      // primero la aplicación
        await _postgres.DisposeAsync(); // después el contenedor
    }
}
```

* `InitializeAsync` arranca el contenedor **antes** de que se construya la aplicación: `ConfigureWebHost` se ejecuta recién al acceder a `Services` o al crear el primer cliente.
* `IAsyncLifetime.DisposeAsync` se implementa de forma **explícita** porque `WebApplicationFactory` ya tiene un `DisposeAsync` público que devuelve `ValueTask`; el de xUnit v2 devuelve `Task`, y con dos firmas iguales el compilador no sabría cuál es cuál.
* Con migraciones, cambia `EnsureCreatedAsync` por `MigrateAsync`: además pruebas que las migraciones se aplican sobre una base vacía.

### 5. El costo y cómo pagarlo una sola vez

Arrancar PostgreSQL tarda unos segundos (más la descarga de la imagen la primera vez). Por eso:

| Alcance | Cómo | Contenedores por ejecución |
| --- | --- | --- |
| Por prueba | Contenedor en el constructor | Uno por prueba: lentísimo |
| Por clase | `IClassFixture<ApiFactory>` | Uno por clase de pruebas |
| Por colección | `ICollectionFixture<ApiFactory>` | Uno compartido por varias clases |

Lo habitual es un contenedor por colección y limpiar los datos entre pruebas ([lección 4](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md)).

### 6. En CI

Los *runners* de GitHub Actions con Linux (`ubuntu-latest`) traen Docker: Testcontainers funciona sin configuración extra. En Azure DevOps, los agentes Linux hospedados también. En agentes Windows o en entornos sin Docker, las pruebas que lo necesitan no pueden correr: sepáralas con un *trait* (`[Trait("Categoria", "Docker")]`) para poder filtrarlas.

-----

## Ejemplo completo

La API de tareas con EF Core y PostgreSQL. `src/Tareas.Api` usa el paquete `Npgsql.EntityFrameworkCore.PostgreSQL` 10.0.0; `tests/Tareas.Api.Tests`, los paquetes `Microsoft.AspNetCore.Mvc.Testing` 10.0.0 y `Testcontainers.PostgreSql` 4.1.0.

`src/Tareas.Api/Program.cs`:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.EntityFrameworkCore;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);

// La cadena se lee al crear el contexto: las pruebas pueden sobrescribirla con UseSetting
builder.Services.AddDbContext<TareasContext>((sp, opciones) =>
    opciones.UseNpgsql(sp.GetRequiredService<IConfiguration>().GetConnectionString("Tareas")));

var app = builder.Build();

var tareas = app.MapGroup("/tareas");

tareas.MapGet("/", async (TareasContext db, CancellationToken ct) =>
    TypedResults.Ok(await db.Tareas
        .OrderBy(t => t.Id)
        .Select(t => new TareaDto(t.Id, t.Titulo, t.Completada))
        .ToListAsync(ct)));

tareas.MapPost("/", async Task<Results<Created<TareaDto>, Conflict<string>, ValidationProblem>> (
    CrearTareaRequest request, TareasContext db, CancellationToken ct) =>
{
    if (string.IsNullOrWhiteSpace(request.Titulo))
    {
        return TypedResults.ValidationProblem(new Dictionary<string, string[]> { ["Titulo"] = ["El título es obligatorio."] });
    }

    var tarea = new Tarea { Titulo = request.Titulo.Trim() };
    db.Tareas.Add(tarea);
    try
    {
        await db.SaveChangesAsync(ct);
    }
    catch (DbUpdateException ex) when (ex.InnerException is PostgresException { SqlState: PostgresErrorCodes.UniqueViolation })
    {
        return TypedResults.Conflict($"Ya existe una tarea con el título '{tarea.Titulo}'.");
    }

    return TypedResults.Created($"/tareas/{tarea.Id}", new TareaDto(tarea.Id, tarea.Titulo, tarea.Completada));
});

app.Run();

public sealed record TareaDto(int Id, string Titulo, bool Completada);
public sealed record CrearTareaRequest(string? Titulo);

public class Tarea
{
    public int Id { get; set; }
    public required string Titulo { get; set; }
    public bool Completada { get; set; }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Tarea> Tareas => Set<Tarea>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Tarea>(tarea =>
        {
            tarea.Property(t => t.Titulo).HasMaxLength(200);
            tarea.HasIndex(t => t.Titulo).IsUnique();
        });
    }
}

public partial class Program { }
```

`tests/Tareas.Api.Tests/ApiFactory.cs`:

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Testcontainers.PostgreSql;

namespace Tareas.Api.Tests;

public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder) =>
        builder.UseSetting("ConnectionStrings:Tareas", _postgres.GetConnectionString());

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<TareasContext>();
        await db.Database.EnsureCreatedAsync();
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await base.DisposeAsync();
        await _postgres.DisposeAsync();
    }
}
```

`tests/Tareas.Api.Tests/TareasPostgresTests.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;

namespace Tareas.Api.Tests;

public class TareasPostgresTests(ApiFactory factory) : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    // Títulos únicos por prueba: comparten la base (la lección 4 resuelve esto con Respawn)
    private static string TituloUnico(string baseTitulo) => $"{baseTitulo} {Guid.NewGuid():N}";

    [Fact]
    public async Task PostTarea_SeGuardaEnPostgresYApareceEnElListado()
    {
        string titulo = TituloUnico("Pagar la luz");

        HttpResponseMessage creada = await _client.PostAsJsonAsync("/tareas", new { titulo });

        Assert.Equal(HttpStatusCode.Created, creada.StatusCode);
        var tareas = await _client.GetFromJsonAsync<List<TareaJson>>("/tareas");
        Assert.Contains(tareas!, t => t.Titulo == titulo);
    }

    [Fact]
    public async Task PostTarea_TituloDuplicado_Devuelve409PorElIndiceUnico()
    {
        string titulo = TituloUnico("Revisar PR");
        await _client.PostAsJsonAsync("/tareas", new { titulo });

        HttpResponseMessage duplicada = await _client.PostAsJsonAsync("/tareas", new { titulo });

        Assert.Equal(HttpStatusCode.Conflict, duplicada.StatusCode);
    }

    [Fact]
    public async Task PostTarea_TituloDe201Caracteres_LoRechazaLaBase()
    {
        string titulo = new('x', 201);

        // La API no valida la longitud: la restricción varchar(200) de PostgreSQL sí
        await Assert.ThrowsAnyAsync<Exception>(async () =>
        {
            HttpResponseMessage respuesta = await _client.PostAsJsonAsync("/tareas", new { titulo });
            respuesta.EnsureSuccessStatusCode();
        });
    }

    private sealed record TareaJson(int Id, string Titulo, bool Completada);
}
```

Ejecución con `dotnet test` (con Docker en marcha). La primera vez descarga la imagen `postgres:17-alpine`; después, arrancar el contenedor tarda unos segundos. Resultado: **3 pruebas superadas, 0 con error**.

Qué observar:

* La prueba del duplicado devuelve `409` porque PostgreSQL **aplicó** el índice único. Con InMemory, la segunda inserción habría devuelto `201` y la prueba habría fallado por la razón contraria: el manejo del duplicado nunca se habría escrito.
* La tercera prueba muestra un hueco real: la API no valida la longitud y la base rechaza el `INSERT`, así que el cliente recibe un `500`. La prueba lo deja documentado; el arreglo es validar en la API (ejercicio 1).
* La API no sabe que está en una prueba: solo recibió otra cadena de conexión por configuración.
* El contenedor se crea una vez para la clase (`IClassFixture<ApiFactory>`) y se destruye al terminar.
* Los títulos con Guid evitan choques entre pruebas que comparten la base. Es un parche: la lección 4 lo resuelve limpiando la base entre pruebas.

-----

## Errores comunes

**1. Probar con el proveedor InMemory y confiar en el resultado.**
Qué pasa: restricciones, traducción a SQL y transacciones que en producción se comportan distinto.
Por qué: InMemory no es una base relacional.
Arreglo: Testcontainers con el mismo motor de producción; SQLite solo si aceptas sus diferencias.

**2. Quitar y volver a registrar el `DbContext` en `ConfigureTestServices`.**
Qué pasa: `InvalidOperationException: Services for database providers 'Microsoft.EntityFrameworkCore.SqlServer', 'Npgsql.EntityFrameworkCore.PostgreSQL' have been registered in the service provider. Only a single database provider can be registered...`.
Por qué: desde EF Core 9, `AddDbContext` también registra la configuración del proveedor por separado (`IDbContextOptionsConfiguration<TContext>`); quitar solo `DbContextOptions<TContext>` deja la original y se suman las dos.
Arreglo: sobrescribe la cadena de conexión por configuración (`UseSetting`); si de verdad cambias de proveedor, quita también esos registros.

**3. Leer la cadena de conexión en una variable al inicio de `Program.cs`.**
Qué pasa: la API sigue conectándose a la base de desarrollo, aunque la prueba configuró otra cadena.
Por qué: la variable se evaluó antes de que la fábrica aplicara su configuración.
Arreglo: leer la cadena dentro del lambda de `AddDbContext`, o usar el patrón de opciones.

**4. Un contenedor por prueba.**
Qué pasa: suites de varios minutos.
Por qué: arrancar un motor de base de datos cuesta segundos.
Arreglo: un contenedor por clase o por colección, y limpieza de datos entre pruebas.

**5. Implementar `DisposeAsync` de forma implícita en una fábrica que también es `IAsyncLifetime`.**
Qué pasa: error de compilación por dos miembros `DisposeAsync` con la misma firma y distinto tipo de retorno.
Por qué: `WebApplicationFactory` ya expone `ValueTask DisposeAsync()`, y `IAsyncLifetime` (v2) pide `Task DisposeAsync()`.
Arreglo: implementación explícita `async Task IAsyncLifetime.DisposeAsync()`. En xUnit v3, `IAsyncLifetime` usa `ValueTask` y hereda de `IAsyncDisposable`.

-----

## Según la versión de .NET

* **EF Core 1.0:** proveedor InMemory, pensado para casos simples y no para simular una base relacional.
* **Testcontainers para .NET 1.x–3.x:** módulos por motor (`Testcontainers.PostgreSql`, `Testcontainers.MsSql`, `Testcontainers.Redis`...).
* **Testcontainers 4 (2024):** limpieza de API y módulos.
* **Documentación actual de EF Core:** desaconseja explícitamente InMemory para probar código de acceso a datos y recomienda el motor real o, como mínimo, SQLite.
* **EF Core 9:** `IDbContextOptionsConfiguration<TContext>`, que cambia cómo se reemplaza un `DbContext` en pruebas.
* **xUnit v3:** `IAsyncLifetime` con `ValueTask`.

-----

## Cuándo sí y cuándo no

**Usa Testcontainers cuando:** el código depende del comportamiento de la base (restricciones, transacciones, SQL específico, migraciones), o de otros servicios con imagen Docker (Redis, RabbitMQ).

**Usa SQLite en memoria cuando:** necesitas pruebas rápidas sin Docker y tu código no depende de particularidades del motor de producción.

**No uses InMemory cuando:** pruebas algo que dependa de la base de datos. Como mucho, sirve para un fake rápido en pruebas que no hablan de persistencia.

-----

## Resumen en 5 líneas

1. Testcontainers arranca el motor real en Docker, con puerto aleatorio, y lo borra al terminar.
2. InMemory no aplica restricciones ni traduce SQL: no sirve para probar acceso a datos.
3. La API lee la cadena por configuración; la prueba la sobrescribe con `UseSetting`.
4. La fábrica implementa `IAsyncLifetime`: arranca el contenedor antes de construir la aplicación.
5. Un contenedor por clase o colección; datos limpios entre pruebas.

-----

## Para profundizar

<details>
<summary>SQL Server en lugar de PostgreSQL</summary>

```csharp
private readonly MsSqlContainer _sql = new MsSqlBuilder()
    .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
    .Build();
```

Con el paquete `Testcontainers.MsSql`. La imagen de SQL Server pesa bastante más y tarda más en arrancar que PostgreSQL. En Macs con Apple Silicon, la imagen corre por emulación; Azure SQL Edge fue una alternativa, pero quedó discontinuada, así que verifica la imagen recomendada para tu plataforma.

</details>

<details>
<summary>Reutilizar contenedores entre ejecuciones locales</summary>

Testcontainers puede marcar un contenedor como reutilizable (`.WithReuse(true)` y habilitar la reutilización en la configuración del usuario) para que no se destruya al terminar y la siguiente ejecución local lo encuentre listo. Ahorra segundos en el ciclo de desarrollo, a cambio de que los datos persistan entre ejecuciones: obliga a limpiar al inicio. En CI no conviene.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para las pruebas de integración con base de datos uso Testcontainers, que levanta un contenedor Docker con el mismo motor que producción y lo borra al terminar. No uso el proveedor InMemory porque no aplica restricciones ni traduce SQL.

### Respuesta ampliada (semi-senior)

Uso Testcontainers con el motor de producción porque InMemory no es relacional y SQLite tiene otra semántica. La fábrica implementa `IAsyncLifetime`: arranca el contenedor antes de construir la app y sobrescribe la cadena con `UseSetting`; la API la lee al crear el contexto, así no tengo que tocar registros (que desde EF Core 9 suma la configuración del proveedor). Aplico migraciones con `MigrateAsync` para probarlas también. Comparto el contenedor por colección y limpio datos entre pruebas, y en CI uso runners Linux con Docker, separando con traits las pruebas que lo requieren.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no InMemory?**
Porque no aplica índices únicos ni FK, no traduce LINQ a SQL y no tiene transacciones reales.

**2. ¿Cómo le pasas la cadena del contenedor a la API?**
Por configuración, con `UseSetting` en la fábrica, y la API la lee al crear el `DbContext`.

**3. ¿No es lento?**
Arrancar el contenedor cuesta unos segundos, por eso se comparte por clase o colección y se limpia con Respawn.

**4. ¿Funciona en CI?**
Sí en runners Linux con Docker, como `ubuntu-latest` en GitHub Actions.

-----

## Práctica

**Ejercicio 1.** La tercera prueba del ejemplo documenta un `500`. Agrega validación de longitud en la API para que responda `400` y reescribe la prueba.

<details>
<summary>Solución</summary>

En el endpoint, junto a la validación del título vacío:

```csharp
if (request.Titulo.Trim().Length > 200)
{
    return TypedResults.ValidationProblem(new Dictionary<string, string[]>
    {
        ["Titulo"] = ["El título no puede superar los 200 caracteres."]
    });
}
```

Prueba:

```csharp
[Fact]
public async Task PostTarea_TituloDe201Caracteres_Devuelve400()
{
    HttpResponseMessage respuesta = await _client.PostAsJsonAsync("/tareas", new { titulo = new string('x', 201) });

    Assert.Equal(HttpStatusCode.BadRequest, respuesta.StatusCode);
}
```

La base sigue teniendo `varchar(200)` como última defensa, pero el cliente ahora recibe un error útil en lugar de un `500`.

</details>

**Ejercicio 2.** Comparte un solo contenedor entre `TareasPostgresTests` y una nueva clase `CategoriasPostgresTests`.

<details>
<summary>Solución</summary>

```csharp
[CollectionDefinition(nameof(ApiCollection))]
public sealed class ApiCollection : ICollectionFixture<ApiFactory> { }

[Collection(nameof(ApiCollection))]
public class TareasPostgresTests(ApiFactory factory) { /* ... */ }

[Collection(nameof(ApiCollection))]
public class CategoriasPostgresTests(ApiFactory factory) { /* ... */ }
```

Se quita `IClassFixture<ApiFactory>` de las clases: la colección crea la fábrica (y su contenedor) una vez y la inyecta en todas. Las clases de una misma colección **no** se ejecutan en paralelo entre sí, lo que evita que se pisen los datos de la base compartida.

</details>

-----

## Siguiente lección

[Datos de prueba y aislamiento con Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md)
