# WebApplicationFactory y pruebas de API en memoria

## En una frase

`WebApplicationFactory<Program>` levanta tu API **completa** (configuración, DI, middleware, routing, serialización) dentro del proceso de pruebas y te da un `HttpClient` para llamarla sin red, de modo que verificas lo que las pruebas unitarias no ven: que todas las piezas funcionan juntas.

-----

## Antes de empezar

Conviene que ya sepas:

* xUnit, fixtures y pruebas asíncronas: [xUnit: Fact, Theory y aserciones](../01-pruebas-unitarias/02-xUnit%20Fact%20Theory%20y%20aserciones.md).
* Minimal APIs y `TypedResults`: [Endpoints y grupos de rutas](../../03-aspnet-core-apis/02-minimal-apis/01-Endpoints%20y%20grupos%20de%20rutas.md).
* Códigos de estado y ProblemDetails: [Errores con ProblemDetails](../../03-aspnet-core-apis/01-diseno-de-apis-rest/02-Errores%20con%20ProblemDetails.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Prueba de integración:** prueba que ejercita varias piezas reales juntas en lugar de una unidad aislada.
* **`WebApplicationFactory<TEntryPoint>`:** clase de `Microsoft.AspNetCore.Mvc.Testing` que arranca la aplicación en memoria para pruebas.
* **`TestServer`:** servidor en memoria que reemplaza a Kestrel durante las pruebas; las peticiones no salen a la red.
* **Punto de entrada (*entry point*):** tipo que identifica el ensamblado de la aplicación; con *top-level statements*, la clase `Program`.
* **Prueba de contrato:** prueba que verifica la forma pública de la API (rutas, códigos, JSON) desde el punto de vista de quien la consume.

-----

## El problema

Una API de tareas tiene 40 pruebas unitarias, todas en verde. Cubren el servicio de tareas con su repositorio mockeado. El primer día en producción:

1. `POST /tareas` responde **415 Unsupported Media Type**: el endpoint quedó esperando un formulario en lugar de JSON por un cambio de atributo. Ninguna prueba unitaria llama al endpoint.
2. `GET /tareas/abc` responde **500**: alguien quitó la restricción `{id:int}` de la ruta y el parámetro se convierte a mano. El servicio nunca recibe `"abc"`, así que sus pruebas no lo ven.
3. El JSON sale con `"Titulo"` en lugar de `"titulo"`: alguien configuró la serialización en `Program.cs` con otra política de nombres. El frontend no muestra ningún título.
4. Un servicio nuevo quedó registrado como *scoped* dentro de un *singleton*: la app no arranca en Development.

Las pruebas unitarias verifican piezas. Ninguna verifica el **cableado**: rutas, binding, serialización, registro en DI, middleware, códigos de estado.

Las consecuencias:

* errores de configuración que solo aparecen al desplegar;
* contratos HTTP que cambian sin que ninguna prueba falle;
* una falsa sensación de seguridad por tener "muchas pruebas".

-----

## Cómo funciona

### 1. Qué levanta `WebApplicationFactory`

```text
Proceso de pruebas (xUnit)
┌──────────────────────────────────────────────────────────────┐
│  prueba ──HttpClient──► TestServer (en memoria, sin sockets)  │
│                           │                                    │
│                           ▼                                    │
│     TU Program.cs real: configuración → DI → middleware        │
│                         → routing → endpoint → JSON            │
└──────────────────────────────────────────────────────────────┘
```

* Ejecuta **tu** `Program.cs`, con tus registros y tu pipeline.
* Reemplaza Kestrel por `TestServer`: no abre puertos y es rápido (milisegundos por petición).
* El entorno es `Development` por defecto, salvo que lo cambies (`builder.UseEnvironment("Testing")`).

### 2. Preparar los proyectos

```text
dotnet new web -o src/Tareas.Api
dotnet new xunit -o tests/Tareas.Api.Tests
dotnet add tests/Tareas.Api.Tests reference src/Tareas.Api
dotnet add tests/Tareas.Api.Tests package Microsoft.AspNetCore.Mvc.Testing
```

`Microsoft.AspNetCore.Mvc.Testing` trae `WebApplicationFactory` y se encarga de detalles como encontrar la carpeta raíz de contenido de la API. Pese al nombre, sirve igual para Minimal APIs.

### 3. La clase `Program` tiene que ser visible

Con *top-level statements*, el compilador genera una clase `Program` interna. El proyecto de pruebas necesita referenciarla como `WebApplicationFactory<Program>`:

```csharp
// Al final de Program.cs
public partial class Program { }
```

En .NET 10, ASP.NET Core genera esta declaración automáticamente para las aplicaciones web; en versiones anteriores hay que escribirla. Agregarla de todos modos no hace daño (es `partial`) y deja explícita la intención.

### 4. Una fábrica compartida por clase

```csharp
public class TareasApiTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetTareas_DevuelveOk()
    {
        HttpResponseMessage respuesta = await _client.GetAsync("/tareas");

        Assert.Equal(HttpStatusCode.OK, respuesta.StatusCode);
    }
}
```

* `IClassFixture` crea **una** fábrica para todas las pruebas de la clase: la aplicación arranca una vez, no por prueba.
* Consecuencia importante: los *singletons* de la API (como un repositorio en memoria) se **comparten** entre esas pruebas. Diseña cada prueba para que no dependa de lo que hicieron las demás (por ejemplo, crea sus propios datos y búscalos por el id que recibió).

### 5. Leer y enviar JSON

`System.Net.Http.Json` agrega métodos de extensión que usan las mismas opciones web que ASP.NET Core (*camelCase*, sin distinguir mayúsculas al leer):

| Método | Para |
| --- | --- |
| `GetFromJsonAsync<T>(url)` | GET y deserializar (lanza si el código no es 2xx) |
| `PostAsJsonAsync(url, objeto)` | POST con cuerpo JSON |
| `respuesta.Content.ReadFromJsonAsync<T>()` | Deserializar una respuesta ya recibida |

Para verificar el código de estado, usa `GetAsync` + `ReadFromJsonAsync`, no `GetFromJsonAsync`: este último lanza una excepción en un `404` y no puedes comprobarlo con un `Assert` claro.

### 6. Qué verificar en una prueba de integración

| Verifica | Ejemplo |
| --- | --- |
| Código de estado | `201`, `400`, `404` |
| Encabezados del contrato | `Location`, `Content-Type: application/problem+json` |
| Forma del JSON | Nombres en *camelCase*, campos esperados |
| Flujos completos | Crear → consultar por la URL del `Location` |
| Configuración | Que la app arranca y los servicios se resuelven |

No repitas aquí todas las combinaciones de reglas de negocio: eso es trabajo de las pruebas unitarias, que son mucho más rápidas. Las de integración verifican el camino HTTP de punta a punta para los casos representativos.

### 7. ¿Usar los DTOs de la API o copias propias?

Si la prueba deserializa con el mismo `TareaDto` de la API, renombrar una propiedad cambia el JSON **y** la prueba a la vez: la prueba sigue verde y el cliente real se rompe. Para pruebas de contrato, usa tipos propios del proyecto de pruebas (o compara el JSON crudo). Para pruebas de flujo, reutilizar los DTOs es aceptable y más simple.

-----

## Ejemplo completo

`src/Tareas.Api/Program.cs`:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<RepositorioTareas>();

var app = builder.Build();

var tareas = app.MapGroup("/tareas");

tareas.MapGet("/", (RepositorioTareas repo) => TypedResults.Ok(repo.Listar()));

tareas.MapGet("/{id:int}", Results<Ok<TareaDto>, NotFound> (int id, RepositorioTareas repo) =>
    repo.Obtener(id) is { } tarea ? TypedResults.Ok(tarea) : TypedResults.NotFound());

tareas.MapPost("/", Results<Created<TareaDto>, ValidationProblem> (CrearTareaRequest request, RepositorioTareas repo) =>
{
    if (string.IsNullOrWhiteSpace(request.Titulo))
    {
        return TypedResults.ValidationProblem(new Dictionary<string, string[]>
        {
            ["Titulo"] = ["El título es obligatorio."]
        });
    }

    TareaDto tarea = repo.Crear(request.Titulo.Trim());
    return TypedResults.Created($"/tareas/{tarea.Id}", tarea);
});

app.Run();

public sealed record TareaDto(int Id, string Titulo, bool Completada);

public sealed record CrearTareaRequest(string? Titulo);

// Singleton con estado: el lock lo hace seguro entre peticiones concurrentes
public sealed class RepositorioTareas
{
    private readonly Lock _lock = new();
    private readonly List<TareaDto> _tareas = [];
    private int _siguienteId = 1;

    public List<TareaDto> Listar()
    {
        lock (_lock) return [.. _tareas];
    }

    public TareaDto? Obtener(int id)
    {
        lock (_lock) return _tareas.FirstOrDefault(t => t.Id == id);
    }

    public TareaDto Crear(string titulo)
    {
        lock (_lock)
        {
            var tarea = new TareaDto(_siguienteId++, titulo, false);
            _tareas.Add(tarea);
            return tarea;
        }
    }
}

public partial class Program { }
```

`tests/Tareas.Api.Tests/TareasApiTests.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc.Testing;

namespace Tareas.Api.Tests;

public class TareasApiTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetTareas_DevuelveOkConUnaLista()
    {
        HttpResponseMessage respuesta = await _client.GetAsync("/tareas");

        Assert.Equal(HttpStatusCode.OK, respuesta.StatusCode);
        var tareas = await respuesta.Content.ReadFromJsonAsync<List<TareaJson>>();
        Assert.NotNull(tareas);
    }

    [Fact]
    public async Task PostTarea_Valida_Devuelve201ConLocationQueSePuedeConsultar()
    {
        HttpResponseMessage creada = await _client.PostAsJsonAsync("/tareas", new { titulo = "Pagar la luz" });

        Assert.Equal(HttpStatusCode.Created, creada.StatusCode);
        Assert.NotNull(creada.Headers.Location);

        HttpResponseMessage consulta = await _client.GetAsync(creada.Headers.Location);
        var tarea = await consulta.Content.ReadFromJsonAsync<TareaJson>();
        Assert.Equal(HttpStatusCode.OK, consulta.StatusCode);
        Assert.Equal("Pagar la luz", tarea!.Titulo);
        Assert.False(tarea.Completada);
    }

    [Fact]
    public async Task PostTarea_SinTitulo_Devuelve400ConProblemDetails()
    {
        HttpResponseMessage respuesta = await _client.PostAsJsonAsync("/tareas", new { titulo = "  " });

        Assert.Equal(HttpStatusCode.BadRequest, respuesta.StatusCode);
        Assert.Equal("application/problem+json", respuesta.Content.Headers.ContentType?.MediaType);
        var problema = await respuesta.Content.ReadFromJsonAsync<HttpValidationProblemDetails>();
        Assert.Contains("Titulo", problema!.Errors.Keys);
    }

    [Fact]
    public async Task GetTarea_Inexistente_Devuelve404()
    {
        HttpResponseMessage respuesta = await _client.GetAsync("/tareas/999999");

        Assert.Equal(HttpStatusCode.NotFound, respuesta.StatusCode);
    }

    [Fact]
    public async Task GetTarea_IdNoNumerico_Devuelve404PorLaRestriccionDeRuta()
    {
        HttpResponseMessage respuesta = await _client.GetAsync("/tareas/abc");

        Assert.Equal(HttpStatusCode.NotFound, respuesta.StatusCode);
    }

    [Fact]
    public async Task GetTareas_JsonUsaCamelCase()
    {
        await _client.PostAsJsonAsync("/tareas", new { titulo = "Revisar PR" });

        string json = await _client.GetStringAsync("/tareas");

        Assert.Contains("\"titulo\":", json);
        Assert.DoesNotContain("\"Titulo\":", json);
    }

    // Tipo propio del proyecto de pruebas: si la API renombra una propiedad, estas pruebas lo detectan
    private sealed record TareaJson(int Id, string Titulo, bool Completada);
}
```

Ejecución con `dotnet test`. Resultado: **6 pruebas superadas, 0 con error** (el formato del resumen depende de la versión del SDK y del idioma).

Qué observar:

* Ninguna prueba conoce `RepositorioTareas`: todas entran por HTTP, como un cliente real.
* La prueba del `201` sigue el `Location` que devolvió la API: verifica que la URL es correcta, no solo que existe.
* `GetTarea_IdNoNumerico` atrapa el error 2 del problema: si alguien quita `{id:int}`, la respuesta cambia.
* `GetTareas_JsonUsaCamelCase` atrapa el error 3: compara el JSON crudo.
* `TareaJson` es una copia propia del contrato: si la API renombra `Titulo`, la deserialización deja la propiedad en `null` y las pruebas fallan.
* Las pruebas comparten la aplicación (y su repositorio *singleton*), así que ninguna asume una lista vacía ni ids concretos. La [lección 4](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md) trata el aislamiento a fondo.

-----

## Errores comunes

**1. `Program` inaccesible.**
Qué pasa: `CS0122: 'Program' is inaccessible due to its protection level` en el proyecto de pruebas.
Por qué: los *top-level statements* generan una clase `Program` interna (en versiones anteriores a .NET 10).
Arreglo: `public partial class Program { }` al final de `Program.cs`.

**2. Usar `GetFromJsonAsync` para probar errores.**
Qué pasa: `HttpRequestException: Response status code does not indicate success: 404 (Not Found)` en lugar de un `Assert` legible.
Por qué: ese método llama a `EnsureSuccessStatusCode`.
Arreglo: `GetAsync` + `Assert.Equal(HttpStatusCode.NotFound, ...)` + `ReadFromJsonAsync`.

**3. Pruebas que dependen del orden o del estado compartido.**
Qué pasa: una prueba que espera "una lista vacía" o "la tarea con id 1" pasa sola y falla con las demás.
Por qué: con `IClassFixture`, todas las pruebas de la clase usan la misma aplicación y sus *singletons*.
Arreglo: cada prueba crea sus datos y los busca por el id o la URL que recibió; o aísla el estado ([lección 4](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md)).

**4. Probar todas las reglas de negocio por HTTP.**
Qué pasa: suites lentas y pruebas de integración que repiten casos de las unitarias.
Por qué: se usa la herramienta más cara para todo.
Arreglo: reglas y combinaciones en unitarias; en integración, el camino HTTP de casos representativos.

**5. Reutilizar los DTOs de la API para pruebas de contrato.**
Qué pasa: un cambio de nombre rompe a los clientes pero no a las pruebas.
Por qué: la API y la prueba cambian juntas.
Arreglo: tipos propios en el proyecto de pruebas o comparación del JSON.

-----

## Según la versión de .NET

* **ASP.NET Core 2.1:** `Microsoft.AspNetCore.Mvc.Testing` y `WebApplicationFactory<TEntryPoint>`.
* **ASP.NET Core 3.0:** basada en el host genérico.
* **ASP.NET Core 6:** soporte para `WebApplication` y *top-level statements* (con `public partial class Program`).
* **.NET 8:** `TimeProvider` y `FakeTimeProvider` facilitan controlar el tiempo en estas pruebas.
* **.NET 10:** ASP.NET Core genera automáticamente la declaración pública de `Program` para que las pruebas la referencien.

-----

## Cuándo sí y cuándo no

**Usa `WebApplicationFactory` cuando:** quieres verificar rutas, binding, validación, serialización, middleware, autenticación o registro en DI, es decir, todo lo que vive entre la petición HTTP y tu lógica.

**Usa pruebas unitarias cuando:** verificas reglas de negocio y sus combinaciones; son órdenes de magnitud más rápidas.

**No uses `WebApplicationFactory` cuando:** necesitas probar comportamiento de Kestrel, TLS real o la red (HTTP/2 real, límites de conexión): eso pide un servidor real o pruebas *end-to-end*.

-----

## Resumen en 5 líneas

1. `WebApplicationFactory<Program>` ejecuta tu `Program.cs` real sobre `TestServer`, sin red.
2. `CreateClient()` da un `HttpClient` para llamar a la API como un cliente real.
3. `IClassFixture` comparte la aplicación entre las pruebas de una clase: no dependas del estado compartido.
4. Verifica códigos, encabezados, forma del JSON y flujos completos; deja las reglas a las unitarias.
5. Para contratos, deserializa con tipos propios o compara el JSON crudo.

-----

## Para profundizar

<details>
<summary>Fábrica propia: subclase de <code>WebApplicationFactory</code></summary>

```csharp
public sealed class ApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.UseSetting("Features:EnviarEmails", "false");
    }
}
```

Centraliza la configuración de pruebas (entorno, ajustes, servicios reemplazados) en una clase y la reutilizas con `IClassFixture<ApiFactory>`. Es la base de las lecciones siguientes.

</details>

<details>
<summary>Ver qué pasa dentro: logs de la API en la salida de la prueba</summary>

Los logs de la aplicación van, por defecto, a la consola del proceso de pruebas, que xUnit no muestra. Con xUnit v3 (o paquetes como `MartinCostello.Logging.XUnit` en v2) puedes enviarlos a `ITestOutputHelper` y verlos junto al resultado de cada prueba. Muy útil cuando una prueba de integración responde `500` y necesitas la excepción.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una prueba de integración prueba varias partes juntas. En ASP.NET Core uso `WebApplicationFactory<Program>`, que levanta la API en memoria, y con `CreateClient()` hago peticiones HTTP reales a mis endpoints para verificar códigos de estado y respuestas.

### Respuesta ampliada (semi-senior)

`WebApplicationFactory` ejecuta el `Program.cs` real sobre `TestServer`, así que verifico el cableado que las unitarias no ven: rutas y restricciones, binding, validación, serialización, middleware y DI. Comparto la fábrica con `IClassFixture` para no arrancar la app por prueba, y por eso diseño pruebas independientes del estado compartido. Verifico códigos, `Location`, `Content-Type` y la forma del JSON, con tipos propios para detectar cambios de contrato. Mantengo pocas pruebas de integración por caso representativo y dejo las combinaciones de reglas a las unitarias.

### Preguntas frecuentes de seguimiento

**1. ¿Las peticiones salen a la red?**
No: `TestServer` las procesa en memoria, sin abrir puertos.

**2. ¿Por qué `public partial class Program`?**
Porque los *top-level statements* generan `Program` como interna y la fábrica necesita referenciarla (en .NET 10 se genera automáticamente).

**3. ¿Una fábrica por prueba o compartida?**
Compartida con `IClassFixture` por velocidad; una nueva cuando la prueba necesita una configuración distinta.

**4. ¿Qué no probarías así?**
Combinaciones de reglas de negocio (van en unitarias) y comportamiento real de red o Kestrel.

-----

## Práctica

**Ejercicio 1.** Agrega a la API `POST /tareas/{id:int}/completar` (`204` o `404`) y escribe dos pruebas de integración: completar una tarea existente y luego verificar `completada: true`, y completar una inexistente.

<details>
<summary>Solución</summary>

En la API:

```csharp
tareas.MapPost("/{id:int}/completar", Results<NoContent, NotFound> (int id, RepositorioTareas repo) =>
    repo.Completar(id) ? TypedResults.NoContent() : TypedResults.NotFound());

// En RepositorioTareas
public bool Completar(int id)
{
    lock (_lock)
    {
        int i = _tareas.FindIndex(t => t.Id == id);
        if (i < 0) return false;
        _tareas[i] = _tareas[i] with { Completada = true };
        return true;
    }
}
```

Pruebas:

```csharp
[Fact]
public async Task Completar_TareaExistente_Devuelve204YQuedaCompletada()
{
    HttpResponseMessage creada = await _client.PostAsJsonAsync("/tareas", new { titulo = "Barrer" });
    Uri location = creada.Headers.Location!;

    HttpResponseMessage respuesta = await _client.PostAsync($"{location}/completar", content: null);

    Assert.Equal(HttpStatusCode.NoContent, respuesta.StatusCode);
    var tarea = await _client.GetFromJsonAsync<TareaJson>(location);
    Assert.True(tarea!.Completada);
}

[Fact]
public async Task Completar_TareaInexistente_Devuelve404()
{
    HttpResponseMessage respuesta = await _client.PostAsync("/tareas/999999/completar", content: null);

    Assert.Equal(HttpStatusCode.NotFound, respuesta.StatusCode);
}
```

Aquí `GetFromJsonAsync` está bien: después del `204`, esperamos un `200`, y si no lo es, que la prueba falle con excepción es aceptable.

</details>

**Ejercicio 2.** Escribe una prueba que verifique que la aplicación **arranca** y que todos los servicios registrados se pueden construir, para detectar dependencias cautivas antes de desplegar.

<details>
<summary>Solución</summary>

```csharp
[Fact]
public void LaAplicacionArrancaConValidacionDeServicios()
{
    using var fabrica = factory.WithWebHostBuilder(b =>
        b.UseDefaultServiceProvider(o =>
        {
            o.ValidateScopes = true;
            o.ValidateOnBuild = true;
        }));

    // Acceder a Services construye el host: si un registro es inválido, lanza aquí
    Assert.NotNull(fabrica.Services);
}
```

`ValidateOnBuild` intenta construir todos los servicios registrados al crear el proveedor; `ValidateScopes` detecta scoped dentro de singletons. En Development ya están activos por defecto, pero esta prueba los fija explícitamente, sin depender del entorno.

</details>

-----

## Siguiente lección

[Reemplazar servicios y autenticación en pruebas](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md)
