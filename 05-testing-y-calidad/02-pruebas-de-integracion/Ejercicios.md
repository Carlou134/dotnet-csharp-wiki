# Ejercicios: pruebas de integración

Requisitos previos: las cuatro lecciones del módulo, la API de tareas de la [lección 4](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md) (PostgreSQL con Testcontainers y Respawn) y Docker en ejecución.

-----

## Ejercicios guiados

### Ejercicio 1. Prueba de humo de todos los GET

**Objetivo:** detectar con una sola prueba parametrizada cualquier endpoint de lectura que responda `500`.

**Contexto:** las pruebas de integración de la lección 1 cubren casos concretos. Una prueba de humo (*smoke test*) es más amplia y superficial: comprueba que cada ruta "no explota". Es barata y atrapa errores de configuración (un servicio sin registrar, una consulta que no se traduce) en endpoints que nadie probó en detalle.

**Instrucciones:**

1. Crea una clase en la colección `ApiCollection` que resetee la base antes de cada prueba.
2. Escribe una `Theory` con las rutas `GET` de la API.
3. Verifica que ninguna responda con un código `5xx`.

<details>
<summary>Solución</summary>

```csharp
using System.Net;

namespace Tareas.Api.Tests;

[Collection(nameof(ApiCollection))]
public class HumoTests(ApiFactory factory) : IAsyncLifetime
{
    private readonly HttpClient _client = factory.CreateClient();

    public Task InitializeAsync() => factory.ResetearBaseAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Theory]
    [InlineData("/tareas")]
    [InlineData("/tareas/999999")]
    public async Task Get_NoRespondeConErrorDeServidor(string ruta)
    {
        HttpResponseMessage respuesta = await _client.GetAsync(ruta);

        Assert.True((int)respuesta.StatusCode < 500,
            $"{ruta} respondió {(int)respuesta.StatusCode}");
    }
}
```

Resultado: **2 pruebas superadas** (una por ruta). `/tareas/999999` responde `404` si existe el endpoint por id, o `405` si no: ambos son `< 500`, porque la prueba solo busca errores de servidor.

</details>

**Qué observar:** el mensaje del `Assert.True` incluye la ruta y el código: si falla, sabes cuál sin depurar. Una prueba de humo no reemplaza a las específicas; es una red gruesa debajo de ellas.

-----

### Ejercicio 2. Completar una tarea y verificar en la base

**Objetivo:** agregar `POST /tareas/{id:int}/completar` con EF Core y verificar el efecto **en la base**, no solo en la respuesta.

**Contexto:** una prueba que solo mira el `204` no detecta un endpoint que responde bien pero olvida llamar a `SaveChangesAsync`. Leer la base desde un scope de la fábrica verifica el efecto real.

**Instrucciones:**

1. Agrega el endpoint a la API con `ExecuteUpdateAsync`.
2. En la prueba, siembra una tarea con `CrearTareaEnBaseAsync`.
3. Llama al endpoint y lee la tarea con un `TareasContext` nuevo.

<details>
<summary>Solución</summary>

En la API:

```csharp
tareas.MapPost("/{id:int}/completar", async Task<Results<NoContent, NotFound>> (int id, TareasContext db, CancellationToken ct) =>
{
    int filas = await db.Tareas
        .Where(t => t.Id == id)
        .ExecuteUpdateAsync(s => s.SetProperty(t => t.Completada, true), ct);
    return filas == 0 ? TypedResults.NotFound() : TypedResults.NoContent();
});
```

En la prueba (clase de la colección `ApiCollection`, con reset en `InitializeAsync`):

```csharp
[Fact]
public async Task Completar_TareaExistente_QuedaCompletadaEnLaBase()
{
    int id = await factory.CrearTareaEnBaseAsync(titulo: "Barrer");

    HttpResponseMessage respuesta = await _client.PostAsync($"/tareas/{id}/completar", content: null);

    Assert.Equal(HttpStatusCode.NoContent, respuesta.StatusCode);

    using var scope = factory.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<TareasContext>();
    Tarea tarea = await db.Tareas.AsNoTracking().SingleAsync(t => t.Id == id);
    Assert.True(tarea.Completada);
}

[Fact]
public async Task Completar_TareaInexistente_Devuelve404()
{
    HttpResponseMessage respuesta = await _client.PostAsync("/tareas/999999/completar", content: null);

    Assert.Equal(HttpStatusCode.NotFound, respuesta.StatusCode);
}
```

Resultado: **2 pruebas superadas**.

</details>

**Qué observar:** el `TareasContext` de la prueba es nuevo (scope propio) y usa `AsNoTracking`: lee lo que realmente quedó en PostgreSQL, no una entidad en memoria.

-----

### Ejercicio 3. Base real + autenticación de prueba

**Objetivo:** combinar la fábrica con Testcontainers y el esquema de autenticación de la [lección 2](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md) para probar un `DELETE` solo para administradores.

**Contexto:** las lecciones mostraron cada técnica por separado. En un proyecto real, la misma fábrica tiene la base del contenedor **y** la autenticación de prueba.

**Instrucciones:**

1. Agrega a la API autenticación JWT y `DELETE /tareas/{id:int}` con `RequireRole("Admin")`.
2. En `ApiFactory.ConfigureWebHost`, registra con `ConfigureTestServices` el esquema `AutenticacionDePrueba` como predeterminado.
3. Escribe pruebas para `401`, `403` y `204`, y verifica que tras el `204` la tarea ya no está en la base.

<details>
<summary>Solución</summary>

En la fábrica:

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.UseSetting("ConnectionStrings:Tareas", _postgres.GetConnectionString());
    builder.ConfigureTestServices(services =>
        services.AddAuthentication(AutenticacionDePrueba.Esquema)
                .AddScheme<AuthenticationSchemeOptions, AutenticacionDePrueba>(AutenticacionDePrueba.Esquema, _ => { }));
}
```

En la API:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer();
builder.Services.AddAuthorization();

tareas.MapDelete("/{id:int}", async Task<Results<NoContent, NotFound>> (int id, TareasContext db, CancellationToken ct) =>
        await db.Tareas.Where(t => t.Id == id).ExecuteDeleteAsync(ct) == 0 ? TypedResults.NotFound() : TypedResults.NoContent())
      .RequireAuthorization(p => p.RequireRole("Admin"));
```

Pruebas:

```csharp
[Theory]
[InlineData(null, HttpStatusCode.Unauthorized)]
[InlineData("Lector", HttpStatusCode.Forbidden)]
[InlineData("Admin", HttpStatusCode.NoContent)]
public async Task Delete_SegunElRol(string? rol, HttpStatusCode esperado)
{
    int id = await factory.CrearTareaEnBaseAsync();
    HttpClient cliente = factory.CreateClient();
    if (rol is not null)
    {
        cliente.DefaultRequestHeaders.Add(AutenticacionDePrueba.EncabezadoRol, rol);
    }

    HttpResponseMessage respuesta = await cliente.DeleteAsync($"/tareas/{id}");

    Assert.Equal(esperado, respuesta.StatusCode);
}

[Fact]
public async Task Delete_ComoAdmin_BorraDeLaBase()
{
    int id = await factory.CrearTareaEnBaseAsync();
    _client.DefaultRequestHeaders.Add(AutenticacionDePrueba.EncabezadoRol, "Admin");

    await _client.DeleteAsync($"/tareas/{id}");

    using var scope = factory.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<TareasContext>();
    Assert.False(await db.Tareas.AnyAsync(t => t.Id == id));
}
```

Resultado: **4 pruebas superadas** (3 casos de la `Theory` + 1 `Fact`).

</details>

**Qué observar:** la `Theory` cubre los tres caminos de seguridad con datos; cada caso tiene la base recién reseteada y su propia tarea. Como la autenticación de prueba está en la fábrica, todas las clases de la colección pueden usarla.

-----

## Retos

### Reto 1. Probar que las migraciones se aplican

**Misión:** reemplaza `EnsureCreatedAsync` por migraciones reales. Crea la migración inicial, aplícala en la fábrica con `MigrateAsync` y escribe una prueba que verifique que la tabla `__EFMigrationsHistory` tiene la migración `InicialCreate` registrada.

**Pista:** `dotnet ef migrations add InicialCreate --project src/Tareas.Api` necesita `Microsoft.EntityFrameworkCore.Design` en la API. Para leer el historial desde la prueba, `db.Database.GetAppliedMigrationsAsync()`. Respawn ya ignora esa tabla.

<details>
<summary>Solución</summary>

En `ApiFactory.InitializeAsync`:

```csharp
using (var scope = Services.CreateScope())
{
    await scope.ServiceProvider.GetRequiredService<TareasContext>().Database.MigrateAsync();
}
```

Prueba:

```csharp
[Fact]
public async Task Migraciones_EstanAplicadas()
{
    using var scope = factory.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<TareasContext>();

    IEnumerable<string> aplicadas = await db.Database.GetAppliedMigrationsAsync();
    IEnumerable<string> pendientes = await db.Database.GetPendingMigrationsAsync();

    Assert.Contains(aplicadas, m => m.EndsWith("_InicialCreate"));
    Assert.Empty(pendientes);
}
```

Los nombres de migración llevan un prefijo con fecha y hora (`20261023..._InicialCreate`), por eso se compara con `EndsWith`. Con esta prueba, una migración que no se puede aplicar sobre una base vacía rompe el build antes de llegar a producción.

</details>

-----

### Reto 2. Pruebas de integración en GitHub Actions

**Misión:** escribe un workflow que ejecute primero las pruebas unitarias y después las de integración (que necesitan Docker), separadas por un *trait*.

**Pista:** marca las clases de integración con `[Trait("Categoria", "Integracion")]` y filtra con `dotnet test --filter`. Los runners `ubuntu-latest` ya traen Docker.

<details>
<summary>Solución</summary>

En las clases de integración:

```csharp
[Collection(nameof(ApiCollection))]
[Trait("Categoria", "Integracion")]
public class TareasAisladasTests(ApiFactory factory) : IAsyncLifetime { /* ... */ }
```

`.github/workflows/pruebas.yml`:

```yaml
name: Pruebas
on: [push, pull_request]

jobs:
  pruebas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Unitarias
        run: dotnet test --filter "Categoria!=Integracion"

      - name: Integración (Docker)
        run: dotnet test --filter "Categoria=Integracion"
```

Separarlas permite fallar rápido: si una unitaria falla, el paso de integración (más lento) ni se ejecuta. Localmente, `dotnet test --filter "Categoria!=Integracion"` deja trabajar sin Docker.

</details>

-----

## Checkpoint

1. ¿Qué verifica una prueba con `WebApplicationFactory` que una prueba unitaria del servicio no puede verificar?
2. ¿Por qué el reemplazo de servicios va en `ConfigureTestServices` y no en `ConfigureServices`?
3. ¿Cómo probarías los caminos `401`, `403` y `200` de un endpoint protegido sin emitir JWT?
4. Nombra tres comportamientos de una base real que el proveedor InMemory de EF Core no reproduce.
5. ¿Por qué una transacción con rollback alrededor de la prueba no aísla pruebas que llaman a la API por HTTP?
6. ¿Por qué Respawn se ejecuta antes de cada prueba y no después?
