# Datos de prueba y aislamiento con Respawn

## En una frase

Las pruebas de integración que comparten una base de datos se pisan entre sí salvo que cada una empiece desde un estado conocido; Respawn deja la base vacía antes de cada prueba en milisegundos, y combinado con un contenedor por colección y datos preparados por la propia prueba, da pruebas rápidas, independientes y repetibles.

-----

## Antes de empezar

Conviene que ya sepas:

* Fábrica con Testcontainers e `IAsyncLifetime`: [Base de datos real con Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md).
* Colecciones y paralelismo de xUnit: [xUnit: Fact, Theory y aserciones](../01-pruebas-unitarias/02-xUnit%20Fact%20Theory%20y%20aserciones.md).
* Transacciones y conexiones: [Transacciones con ADO.NET, Dapper y EF Core](../../04-data-access-and-transactions/02-ado-net-y-dapper/04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Aislamiento de pruebas:** que el resultado de una prueba no dependa de lo que hicieron otras.
* **Respawn:** biblioteca que borra los datos de una base respetando las claves foráneas, sin recrear el esquema.
* **Estado conocido:** contenido exacto de la base al empezar una prueba (por ejemplo, vacía, o con ciertos datos de referencia).
* **Colección de xUnit:** grupo de clases de prueba que comparten un fixture y no se ejecutan en paralelo entre sí.
* **Builder de datos de prueba:** helper que crea datos válidos por defecto y permite cambiar solo lo que la prueba necesita.

-----

## El problema

Después de la lección 3, la suite tiene dos clases que comparten el contenedor:

```csharp
[Fact]
public async Task GetTareas_SinDatos_DevuelveListaVacia()
{
    var tareas = await _client.GetFromJsonAsync<List<TareaJson>>("/tareas");
    Assert.Empty(tareas!);
}

[Fact]
public async Task PostTarea_DosVeces_ElListadoTieneDos()
{
    await _client.PostAsJsonAsync("/tareas", new { titulo = "A" });
    await _client.PostAsJsonAsync("/tareas", new { titulo = "B" });
    var tareas = await _client.GetFromJsonAsync<List<TareaJson>>("/tareas");
    Assert.Equal(2, tareas!.Count);
}
```

Lo que pasa:

1. Cada una pasa **sola**. Juntas, depende del orden: si la segunda corre primero, la primera encuentra dos tareas y falla.
2. Al volver a correr la suite, la segunda falla con `409`: "A" ya existe de la ejecución anterior... si el contenedor se reutilizó.
3. Para "arreglarlo", alguien agrega Guids a todos los títulos y cambia `Assert.Equal(2, ...)` por `Assert.True(tareas.Count >= 2)`. Las pruebas pasan, pero ya no verifican lo que decían verificar.
4. Otro intenta recrear la base (`EnsureDeleted` + `EnsureCreated`) antes de cada prueba: la suite pasa de 10 segundos a 3 minutos.

Las consecuencias:

* pruebas que fallan según el orden o la ejecución anterior (*flaky tests*);
* aserciones debilitadas para convivir con datos ajenos;
* suites lentas que nadie quiere ejecutar.

-----

## Cómo funciona

### 1. Estrategias de aislamiento

| Estrategia | Cómo | Velocidad | Problema |
| --- | --- | --- | --- |
| Datos únicos (Guids) | Cada prueba usa valores que nadie más usa | Rápida | No sirve para contar ni para "lista vacía"; las aserciones se debilitan |
| Recrear la base | `EnsureDeleted` + `EnsureCreated`/`Migrate` por prueba | Muy lenta | Segundos por prueba |
| Transacción con rollback | Abrir transacción, ejecutar, deshacer | Rápida | **No funciona** a través de HTTP: la API usa sus propios `DbContext` y conexiones |
| **Borrar datos (Respawn)** | Vaciar las tablas antes de cada prueba | Rápida (milisegundos) | Pruebas de una misma base en secuencia |
| Base por clase o colección | Una base (o contenedor) distinta por grupo | Media | Más recursos; combinable con Respawn |

Para pruebas de API, la combinación habitual es **un contenedor por colección + Respawn antes de cada prueba**.

### 2. Respawn

```csharp
_respawner = await Respawner.CreateAsync(conexion, new RespawnerOptions
{
    DbAdapter = DbAdapter.Postgres,
    SchemasToInclude = ["public"],
    TablesToIgnore = [new Table("__EFMigrationsHistory")]
});

await _respawner.ResetAsync(conexion);   // antes de cada prueba
```

* `CreateAsync` lee el esquema **una vez**: tablas y claves foráneas.
* `ResetAsync` borra los datos en el orden correcto (primero las hijas, después las padres) para no violar FKs. No toca el esquema.
* Ignora la tabla de historial de migraciones: si la vaciaras, la próxima `MigrateAsync` intentaría aplicar todo de nuevo.
* Tiene adaptadores para SQL Server, PostgreSQL, MySQL, Oracle e Informix.

### 3. Dónde llamar a `ResetAsync`: antes de cada prueba

```csharp
[Collection(nameof(ApiCollection))]
public class TareasTests(ApiFactory factory) : IAsyncLifetime
{
    public Task InitializeAsync() => factory.ResetearBaseAsync();   // antes de CADA prueba
    public Task DisposeAsync() => Task.CompletedTask;
}
```

xUnit crea una instancia de la clase por prueba y llama a su `InitializeAsync` antes de ejecutarla. Resetear **antes** (y no después) garantiza el estado inicial aunque la prueba anterior haya fallado a mitad de camino o el proceso se haya cortado.

### 4. Colección: un contenedor, sin paralelismo entre clases

```csharp
[CollectionDefinition(nameof(ApiCollection))]
public sealed class ApiCollection : ICollectionFixture<ApiFactory> { }
```

```text
Colección "ApiCollection" (un ApiFactory, un contenedor PostgreSQL)
  TareasTests ────────┐
  CategoriasTests ────┼──► se ejecutan en SECUENCIA: nunca dos pruebas sobre la base a la vez
  UsuariosTests ──────┘
Otras colecciones (pruebas unitarias, otro contenedor) ──► en paralelo con esta
```

Si dos clases usaran la base a la vez, el `ResetAsync` de una borraría los datos que la otra acaba de crear.

### 5. Datos de prueba: la prueba prepara lo que necesita

Con la base vacía, cada prueba arma su escenario. Dos formas:

| Forma | Ejemplo | Ventaja |
| --- | --- | --- |
| Por la API | `await _client.PostAsJsonAsync("/tareas", ...)` | Ejerce el camino real; no hay atajos |
| Directo en la base | `db.Tareas.Add(...)` desde un scope de la fábrica | Rápido para preparar muchos datos o estados difíciles de alcanzar por la API |

```csharp
public async Task<int> CrearTareaEnBaseAsync(string titulo = "Tarea de prueba", bool completada = false)
{
    using var scope = Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<TareasContext>();
    var tarea = new Tarea { Titulo = titulo, Completada = completada };
    db.Tareas.Add(tarea);
    await db.SaveChangesAsync();
    return tarea.Id;
}
```

Los parámetros con valores por defecto son un *builder* mínimo: la prueba solo indica lo que le importa (`completada: true`) y el resto queda válido.

### 6. Datos de referencia

Si la API necesita datos fijos para funcionar (categorías, países, roles), tienes dos opciones:

* Ignorar esas tablas en Respawn (`TablesToIgnore`) y sembrarlas una vez al iniciar la fábrica (con `HasData` aparecen solas al crear el esquema).
* Volver a sembrarlas después de cada `ResetAsync`.

La primera es más rápida; la segunda, más segura si alguna prueba puede modificar esos datos.

### 7. No dependas de ids

Respawn borra filas, pero no reinicia las secuencias de los ids: después de un reset, la siguiente tarea puede tener el id 7. Las pruebas usan el id o el `Location` que devuelve la API, nunca un número fijo.

-----

## Ejemplo completo

La API de la [lección 3](03-Base%20de%20datos%20real%20con%20Testcontainers.md) sin cambios. El proyecto de pruebas agrega `Respawn` 6.2.1 (y usa `Npgsql`, que ya llega por la referencia a la API).

`tests/Tareas.Api.Tests/ApiFactory.cs`:

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Npgsql;
using Respawn;
using Respawn.Graph;
using Testcontainers.PostgreSql;

namespace Tareas.Api.Tests;

public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .Build();

    private NpgsqlConnection _conexion = null!;
    private Respawner _respawner = null!;

    protected override void ConfigureWebHost(IWebHostBuilder builder) =>
        builder.UseSetting("ConnectionStrings:Tareas", _postgres.GetConnectionString());

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        using (var scope = Services.CreateScope())
        {
            await scope.ServiceProvider.GetRequiredService<TareasContext>().Database.EnsureCreatedAsync();
        }

        _conexion = new NpgsqlConnection(_postgres.GetConnectionString());
        await _conexion.OpenAsync();
        _respawner = await Respawner.CreateAsync(_conexion, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"],
            TablesToIgnore = [new Table("__EFMigrationsHistory")]
        });
    }

    public Task ResetearBaseAsync() => _respawner.ResetAsync(_conexion);

    public async Task<int> CrearTareaEnBaseAsync(string titulo = "Tarea de prueba", bool completada = false)
    {
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<TareasContext>();
        var tarea = new Tarea { Titulo = titulo, Completada = completada };
        db.Tareas.Add(tarea);
        await db.SaveChangesAsync();
        return tarea.Id;
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await _conexion.DisposeAsync();
        await base.DisposeAsync();
        await _postgres.DisposeAsync();
    }
}

[CollectionDefinition(nameof(ApiCollection))]
public sealed class ApiCollection : ICollectionFixture<ApiFactory> { }
```

`tests/Tareas.Api.Tests/TareasAisladasTests.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;

namespace Tareas.Api.Tests;

[Collection(nameof(ApiCollection))]
public class TareasAisladasTests(ApiFactory factory) : IAsyncLifetime
{
    private readonly HttpClient _client = factory.CreateClient();

    public Task InitializeAsync() => factory.ResetearBaseAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task GetTareas_SinDatos_DevuelveListaVacia()
    {
        var tareas = await _client.GetFromJsonAsync<List<TareaJson>>("/tareas");

        Assert.Empty(tareas!);
    }

    [Fact]
    public async Task PostTarea_DosVeces_ElListadoTieneExactamenteDos()
    {
        await _client.PostAsJsonAsync("/tareas", new { titulo = "A" });
        await _client.PostAsJsonAsync("/tareas", new { titulo = "B" });

        var tareas = await _client.GetFromJsonAsync<List<TareaJson>>("/tareas");

        Assert.Equal(new[] { "A", "B" }, tareas!.Select(t => t.Titulo));
    }

    [Fact]
    public async Task PostTarea_TituloDuplicado_Devuelve409()
    {
        await _client.PostAsJsonAsync("/tareas", new { titulo = "A" });

        HttpResponseMessage duplicada = await _client.PostAsJsonAsync("/tareas", new { titulo = "A" });

        Assert.Equal(HttpStatusCode.Conflict, duplicada.StatusCode);
    }

    [Fact]
    public async Task GetTareas_ConTareaSembradaEnLaBase_LaDevuelveCompletada()
    {
        await factory.CrearTareaEnBaseAsync(titulo: "Sembrada", completada: true);

        var tareas = await _client.GetFromJsonAsync<List<TareaJson>>("/tareas");

        TareaJson unica = Assert.Single(tareas!);
        Assert.Equal("Sembrada", unica.Titulo);
        Assert.True(unica.Completada);
    }

    private sealed record TareaJson(int Id, string Titulo, bool Completada);
}
```

Ejecución con `dotnet test` (con Docker en marcha). Resultado: **4 pruebas superadas, 0 con error**, en cualquier orden y en ejecuciones repetidas.

Qué observar:

* Las aserciones son **exactas** (`Assert.Empty`, la lista `{ "A", "B" }`, `Assert.Single`): con la base vacía al inicio, no hace falta debilitarlas.
* Las pruebas 2 y 3 usan el mismo título "A" sin chocar, porque Respawn vació la tabla entre una y otra.
* `CrearTareaEnBaseAsync` prepara un estado (tarea ya completada) que por la API requeriría dos llamadas; la verificación sí pasa por la API.
* `_conexion` queda abierta durante toda la colección y solo la usa Respawn; la API usa sus propias conexiones.
* Si agregas otra clase con `[Collection(nameof(ApiCollection))]`, comparte el contenedor y se ejecuta en secuencia con esta.

-----

## Errores comunes

**1. Debilitar aserciones para convivir con datos ajenos.**
Qué pasa: `Count >= 2` o `Contains` donde se quería `Equal`; las pruebas dejan de detectar duplicados o filas de más.
Por qué: la base no empieza en un estado conocido.
Arreglo: Respawn antes de cada prueba y aserciones exactas.

**2. Intentar aislar con una transacción alrededor de la prueba.**
Qué pasa: los datos creados por la API quedan confirmados igual, o la API no ve los datos que preparó la prueba.
Por qué: la prueba y la API usan conexiones y `DbContext` distintos; una transacción local no los abarca.
Arreglo: borrar datos con Respawn, o base por grupo de pruebas.

**3. Resetear después de cada prueba en lugar de antes.**
Qué pasa: si una prueba falla o se interrumpe antes del reset, la siguiente arranca con datos sucios.
Por qué: el "después" no siempre llega.
Arreglo: `ResetAsync` en `InitializeAsync`, antes de la prueba.

**4. Vaciar la tabla de migraciones.**
Qué pasa: `MigrateAsync` intenta reaplicar todas las migraciones y falla con "la tabla ya existe".
Por qué: el historial de migraciones también se borró.
Arreglo: `TablesToIgnore = [new Table("__EFMigrationsHistory")]`.

**5. Clases que comparten la base en colecciones distintas.**
Qué pasa: fallos intermitentes: el reset de una borra los datos de la otra a mitad de prueba.
Por qué: xUnit ejecuta colecciones distintas en paralelo.
Arreglo: misma colección para todo lo que usa la misma base, o una base por colección.

**6. Ids fijos en las pruebas.**
Qué pasa: `GET /tareas/1` devuelve `404` después del primer reset.
Por qué: Respawn no reinicia las secuencias.
Arreglo: usar el id o el `Location` devuelto por la creación.

-----

## Según la versión de .NET

* **Respawn 1–5:** `Checkpoint` como clase principal.
* **Respawn 6 (2022):** `Respawner.CreateAsync` + `ResetAsync`, opciones en `RespawnerOptions` y tipo `Table` para tablas a incluir o ignorar.
* **xUnit 2:** colecciones con `ICollectionFixture<T>` e `IAsyncLifetime` en clases de prueba y fixtures.
* **xUnit v3:** fixtures de ensamblado (`AssemblyFixture`) para compartir un recurso entre todas las colecciones.

-----

## Cuándo sí y cuándo no

**Usa Respawn cuando:** varias pruebas comparten una base real y necesitan empezar desde un estado conocido.

**Usa datos únicos (Guids) cuando:** las pruebas solo buscan "su" dato y nunca cuentan ni esperan una tabla vacía; o como complemento en pruebas que corren en paralelo sobre la misma base.

**Usa una base por colección cuando:** quieres paralelizar grupos grandes de pruebas de integración y tienes recursos para varios contenedores.

-----

## Resumen en 5 líneas

1. Pruebas que comparten base necesitan empezar desde un estado conocido.
2. Respawn vacía las tablas respetando FKs en milisegundos, sin recrear el esquema.
3. Resetea **antes** de cada prueba, en el `InitializeAsync` de la clase.
4. Un contenedor por colección: las clases de la colección corren en secuencia.
5. Cada prueba prepara sus datos (por la API o directo en la base) y usa los ids que recibe.

-----

## Para profundizar

<details>
<summary>Paralelizar con una base por colección</summary>

En lugar de un contenedor por colección, puedes usar un contenedor y crear una **base** distinta por colección (`CREATE DATABASE tareas_coleccion_2`), con su propia cadena de conexión. Cada colección corre en paralelo con las demás, cada una con su Respawn. Es más complejo de armar; vale la pena cuando la suite de integración crece a cientos de pruebas.

</details>

<details>
<summary>Builders de datos de prueba más completos</summary>

Cuando las entidades tienen muchas propiedades, un *builder* fluido (`new TareaBuilder().ConTitulo("X").Completada().Construir()`) o bibliotecas como Bogus (datos falsos realistas) evitan repetir objetos enormes en cada prueba. La regla sigue siendo la misma: valores válidos por defecto y la prueba cambia solo lo que le importa, para que se lea qué escenario está probando.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para que las pruebas de integración no se pisen, uso Respawn, que vacía la base antes de cada prueba. Comparto un contenedor por colección de xUnit para que las clases que usan la misma base no corran a la vez.

### Respuesta ampliada (semi-senior)

Combino un contenedor por colección (`ICollectionFixture`) con Respawn en el `InitializeAsync` de cada clase, así cada prueba arranca con la base vacía y puedo usar aserciones exactas. Respawn lee el esquema una vez y borra en el orden de las FKs; ignoro el historial de migraciones y siembro aparte los datos de referencia. Descarto la transacción con rollback porque la API usa sus propias conexiones. Cada prueba prepara sus datos por la API o con helpers que escriben en la base, y nunca dependo de ids fijos. Para escalar, una base por colección permite paralelizar.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no recrear la base en cada prueba?**
Porque tarda segundos por prueba; Respawn tarda milisegundos.

**2. ¿Por qué no una transacción con rollback?**
Porque la API abre sus propias conexiones y la transacción de la prueba no las abarca.

**3. ¿Reset antes o después?**
Antes: garantiza el estado inicial aunque la prueba anterior haya fallado.

**4. ¿Cómo evitas que dos clases se pisen?**
Poniéndolas en la misma colección de xUnit, que no ejecuta en paralelo.

-----

## Práctica

**Ejercicio 1.** Agrega una tabla `Categorias` con `HasData` (dos categorías fijas) y configura Respawn para que no la borre. Escribe una prueba que verifique que siguen ahí después de un reset.

<details>
<summary>Solución</summary>

En la fábrica:

```csharp
_respawner = await Respawner.CreateAsync(_conexion, new RespawnerOptions
{
    DbAdapter = DbAdapter.Postgres,
    SchemasToInclude = ["public"],
    TablesToIgnore = [new Table("__EFMigrationsHistory"), new Table("Categorias")]
});
```

Prueba (suponiendo `GET /categorias`):

```csharp
[Fact]
public async Task Categorias_SobrevivenAlReset()
{
    var categorias = await _client.GetFromJsonAsync<List<CategoriaJson>>("/categorias");

    Assert.Equal(2, categorias!.Count);
}

private sealed record CategoriaJson(int Id, string Nombre);
```

Como el `InitializeAsync` de la clase ya ejecutó un reset antes de esta prueba, que haya exactamente dos categorías demuestra que Respawn las ignoró. Ojo: si alguna prueba modifica categorías, esos cambios persisten entre pruebas; en ese caso conviene resembrarlas después de cada reset.

</details>

**Ejercicio 2.** Una prueba falla de forma intermitente: "esperaba 1 tarea, encontré 0". Está en una clase con `[Collection("Base")]` y otra clase usa la misma fábrica con `IClassFixture<ApiFactory>`. ¿Qué pasa?

<details>
<summary>Solución</summary>

Con `IClassFixture<ApiFactory>`, la segunda clase crea **su propia** fábrica (y su propio contenedor), así que no comparten base... salvo que la fábrica no use un contenedor propio y ambas apunten a la misma base externa. Si comparten base, la causa es que pertenecen a colecciones distintas (la clase con `IClassFixture` forma su propia colección implícita) y xUnit las ejecuta **en paralelo**: el `ResetAsync` de una borra la tarea que la otra acaba de crear. El arreglo es poner ambas en la misma colección (`[Collection("Base")]`, quitando `IClassFixture`) para que corran en secuencia, o darle a cada una su propia base.

</details>

-----

## Siguiente lección

Terminaste el módulo. Practica con los [Ejercicios del módulo](Ejercicios.md) y vuelve al [índice](README.md).
