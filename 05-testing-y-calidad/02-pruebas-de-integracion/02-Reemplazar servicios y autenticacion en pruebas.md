# Reemplazar servicios y autenticación en pruebas

## En una frase

`ConfigureTestServices` registra, **después** de los registros de la API, las versiones de prueba de lo que no quieres real (pasarela de pago, email, reloj, autenticación); así pruebas endpoints protegidos con usuarios y roles a elección y comportamientos que dependen de la fecha, sin tocar `Program.cs`.

-----

## Antes de empezar

Conviene que ya sepas:

* `WebApplicationFactory` y pruebas por HTTP: [WebApplicationFactory y pruebas de API en memoria](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md).
* Esquemas de autenticación, `401` y `403`: [Autenticación JWT bearer](../../03-aspnet-core-apis/03-seguridad-de-apis/02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md) y [Autorización con claims y políticas](../../03-aspnet-core-apis/03-seguridad-de-apis/03-Autorizacion%20con%20claims%20y%20politicas.md).
* Registros múltiples en DI ("gana el último"): [Inyección de dependencias y ciclos de vida](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`ConfigureTestServices`:** método de `IWebHostBuilder` que agrega registros de servicios después de los de la aplicación.
* **`WithWebHostBuilder`:** crea una fábrica derivada con configuración extra, sin modificar la original.
* **Handler de autenticación de prueba:** esquema de autenticación que construye un usuario a partir de datos de la petición, solo para pruebas.
* **`FakeTimeProvider`:** implementación de `TimeProvider` cuya hora controla la prueba (`Microsoft.Extensions.TimeProvider.Testing`).

-----

## El problema

La API de tareas agrega dos cosas:

* `DELETE /tareas/{id}` solo para administradores (JWT con rol `Admin`).
* `GET /tareas/vencidas`, que compara la fecha de vencimiento con "hoy".

El equipo intenta probarlas:

```csharp
// Generar un JWT real para cada prueba...
var token = GenerarTokenConClaveDeProduccion("Admin");   // ¿con qué clave? ¿la de producción?
_client.DefaultRequestHeaders.Authorization = new("Bearer", token);

// ...y para "vencidas", crear tareas que vencieron "ayer"
await _client.PostAsJsonAsync("/tareas", new { titulo = "X", vence = DateOnly.FromDateTime(DateTime.Today.AddDays(-1)) });
```

Lo que pasa:

1. Para firmar tokens, alguien copia la clave de producción al proyecto de pruebas. Ahora está en el repositorio.
2. Las pruebas de "vencidas" usan `DateTime.Today` de la máquina, mientras la API usa la hora UTC del servidor. Entre las 21:00 y la medianoche (en UTC−3), "hoy" no es el mismo día para ambos, y la prueba falla **solo de noche**.
3. Nadie prueba el `401` ni el `403`: generar tokens inválidos a propósito era demasiado trabajo.

Las consecuencias:

* secretos de producción en el código de pruebas;
* pruebas que dependen de la hora en que corren;
* los caminos de seguridad más importantes, sin probar.

-----

## Cómo funciona

### 1. `ConfigureTestServices`: registrar después de la API

```text
Program.cs (API)                         ConfigureTestServices (prueba)
AddSingleton<TimeProvider>(System)   →   AddSingleton<TimeProvider>(fakeReloj)
AddAuthentication("Bearer")          →   AddAuthentication("Test").AddScheme<..., AutenticacionDePrueba>
AddScoped<IPasarelaPago, Stripe>     →   AddScoped<IPasarelaPago, PasarelaFalsa>
                                         ▲ se registran DESPUÉS: al pedir un servicio, gana el último
```

```csharp
var cliente = factory.WithWebHostBuilder(builder =>
    builder.ConfigureTestServices(services =>
    {
        services.AddSingleton<TimeProvider>(reloj);
        services.AddScoped<IPasarelaPago, PasarelaFalsa>();
    }))
    .CreateClient();
```

* Para servicios que se piden de a uno, el último registro gana: no hace falta quitar el original.
* Si el servicio se pide como `IEnumerable<T>` (varios handlers), sí hay que quitar el original: `services.RemoveAll<IPasarelaPago>()` antes de agregar el falso.
* `ConfigureTestServices` corre después de `Program.cs`; `ConfigureServices` (sin "Test") corre **antes**, y la API sobrescribiría tus reemplazos.

### 2. `WithWebHostBuilder`: una fábrica por configuración

Cada llamada crea una **aplicación nueva** (con sus propios singletons). Es más lenta que compartir la fábrica, pero cada prueba arranca con su propia configuración y su propio estado. La fábrica original se encarga de descartar las derivadas.

| | `factory.CreateClient()` | `factory.WithWebHostBuilder(...).CreateClient()` |
| --- | --- | --- |
| Aplicación | Compartida | Nueva |
| Estado en memoria | Compartido entre pruebas | Propio |
| Velocidad | Arranca una vez | Arranca en cada llamada |
| Cuándo | Configuración común | Configuración distinta por prueba |

### 3. Autenticación de prueba: un handler propio

En lugar de fabricar JWT, reemplaza el esquema de autenticación por uno que construye el usuario a partir de un encabezado:

```csharp
public sealed class AutenticacionDePrueba(
    IOptionsMonitor<AuthenticationSchemeOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string Esquema = "Test";
    public const string EncabezadoRol = "X-Test-Rol";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(EncabezadoRol, out var rol))
        {
            return Task.FromResult(AuthenticateResult.NoResult());   // anónimo → 401 en endpoints protegidos
        }

        var identidad = new ClaimsIdentity(
            [new Claim("sub", "usuario-de-prueba"), new Claim(ClaimTypes.Role, rol.ToString())],
            Esquema);
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identidad), Esquema);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

```csharp
services.AddAuthentication(AutenticacionDePrueba.Esquema)
        .AddScheme<AuthenticationSchemeOptions, AutenticacionDePrueba>(AutenticacionDePrueba.Esquema, _ => { });
```

* `AddAuthentication("Test")` cambia el **esquema por defecto**: la autorización autentica con el handler de prueba en lugar de JWT.
* Sin encabezado: `NoResult()` → el endpoint protegido responde `401`. Con un rol sin permiso: `403`. Con el rol correcto: pasa.
* Lo que **no** prueba: la validación del JWT (firma, emisor, audiencia). Eso lo garantiza el handler de ASP.NET Core y tu configuración; si quieres cubrir la configuración, agrega aparte una o dos pruebas con tokens reales firmados con una clave **de prueba**.

### 4. Controlar el tiempo con `FakeTimeProvider`

La API pide `TimeProvider` en lugar de usar `DateTime.UtcNow`:

```csharp
builder.Services.AddSingleton(TimeProvider.System);

app.MapGet("/tareas/vencidas", (RepositorioTareas repo, TimeProvider reloj) =>
    TypedResults.Ok(repo.Vencidas(DateOnly.FromDateTime(reloj.GetUtcNow().UtcDateTime))));
```

La prueba fija "hoy":

```csharp
var reloj = new FakeTimeProvider(new DateTimeOffset(2026, 3, 15, 12, 0, 0, TimeSpan.Zero));
// reloj.Advance(TimeSpan.FromDays(1)) si necesitas que pase el tiempo dentro de la prueba
```

La prueba da el mismo resultado a cualquier hora, en cualquier zona horaria y en cualquier máquina.

### 5. Qué reemplazar en integración (y qué no)

| Reemplaza | Por qué |
| --- | --- |
| Servicios externos: pagos, email, SMS, APIs de terceros | Costo, efectos reales, red inestable |
| Reloj (`TimeProvider`) | Repetibilidad |
| Autenticación (esquema de prueba) | Usuarios y roles a elección, sin secretos |

| No reemplaces | Por qué |
| --- | --- |
| Tu dominio, tus servicios y tus validaciones | Es justamente lo que la prueba de integración debe ejercer |
| La base de datos (si puedes) | La [lección 3](03-Base%20de%20datos%20real%20con%20Testcontainers.md) usa una real |
| Middleware y autorización | El objetivo es probar el pipeline real |

Cuantos más servicios reemplazas, más se parece la prueba a una unitaria cara.

-----

## Ejemplo completo

La API de la [lección 1](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md), con autenticación, un endpoint para administradores y otro que depende de la fecha. `src/Tareas.Api` agrega el paquete `Microsoft.AspNetCore.Authentication.JwtBearer` 10.0.0; `tests/Tareas.Api.Tests`, el paquete `Microsoft.Extensions.TimeProvider.Testing` 10.0.0.

`src/Tareas.Api/Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<RepositorioTareas>();
builder.Services.AddSingleton(TimeProvider.System);

// En producción, la configuración sale de Authentication:Schemes:Bearer (emisor, audiencia, claves)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();

var tareas = app.MapGroup("/tareas");

tareas.MapPost("/", Results<Created<TareaDto>, ValidationProblem> (CrearTareaRequest request, RepositorioTareas repo) =>
{
    if (string.IsNullOrWhiteSpace(request.Titulo))
    {
        return TypedResults.ValidationProblem(new Dictionary<string, string[]> { ["Titulo"] = ["El título es obligatorio."] });
    }
    TareaDto tarea = repo.Crear(request.Titulo.Trim(), request.Vence);
    return TypedResults.Created($"/tareas/{tarea.Id}", tarea);
});

tareas.MapGet("/vencidas", (RepositorioTareas repo, TimeProvider reloj) =>
    TypedResults.Ok(repo.Vencidas(DateOnly.FromDateTime(reloj.GetUtcNow().UtcDateTime))));

tareas.MapDelete("/{id:int}", Results<NoContent, NotFound> (int id, RepositorioTareas repo) =>
        repo.Eliminar(id) ? TypedResults.NoContent() : TypedResults.NotFound())
      .RequireAuthorization(p => p.RequireRole("Admin"));

app.Run();

public sealed record TareaDto(int Id, string Titulo, DateOnly? Vence, bool Completada);

public sealed record CrearTareaRequest(string? Titulo, DateOnly? Vence);

public sealed class RepositorioTareas
{
    private readonly Lock _lock = new();
    private readonly List<TareaDto> _tareas = [];
    private int _siguienteId = 1;

    public TareaDto Crear(string titulo, DateOnly? vence)
    {
        lock (_lock)
        {
            var tarea = new TareaDto(_siguienteId++, titulo, vence, false);
            _tareas.Add(tarea);
            return tarea;
        }
    }

    public List<TareaDto> Vencidas(DateOnly hoy)
    {
        lock (_lock) return _tareas.Where(t => !t.Completada && t.Vence < hoy).OrderBy(t => t.Vence).ToList();
    }

    public bool Eliminar(int id)
    {
        lock (_lock) return _tareas.RemoveAll(t => t.Id == id) > 0;
    }
}

public partial class Program { }
```

`tests/Tareas.Api.Tests/AutenticacionDePrueba.cs`:

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace Tareas.Api.Tests;

public sealed class AutenticacionDePrueba(
    IOptionsMonitor<AuthenticationSchemeOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string Esquema = "Test";
    public const string EncabezadoRol = "X-Test-Rol";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(EncabezadoRol, out var rol))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        var identidad = new ClaimsIdentity(
            [new Claim("sub", "usuario-de-prueba"), new Claim(ClaimTypes.Role, rol.ToString())],
            Esquema);
        return Task.FromResult(AuthenticateResult.Success(
            new AuthenticationTicket(new ClaimsPrincipal(identidad), Esquema)));
    }
}
```

`tests/Tareas.Api.Tests/TareasProtegidasTests.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Time.Testing;

namespace Tareas.Api.Tests;

public class TareasProtegidasTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    // Cada cliente tiene su propia aplicación: estado y reloj propios
    private HttpClient CrearCliente(TimeProvider? reloj = null) =>
        factory.WithWebHostBuilder(builder => builder.ConfigureTestServices(services =>
        {
            services.AddAuthentication(AutenticacionDePrueba.Esquema)
                    .AddScheme<AuthenticationSchemeOptions, AutenticacionDePrueba>(AutenticacionDePrueba.Esquema, _ => { });
            if (reloj is not null)
            {
                services.AddSingleton(reloj);
            }
        }))
        .CreateClient();

    private static async Task<Uri> CrearTareaAsync(HttpClient cliente, string titulo, DateOnly? vence = null)
    {
        HttpResponseMessage respuesta = await cliente.PostAsJsonAsync("/tareas", new { titulo, vence });
        respuesta.EnsureSuccessStatusCode();
        return respuesta.Headers.Location!;
    }

    [Fact]
    public async Task Delete_SinAutenticar_Devuelve401()
    {
        HttpClient cliente = CrearCliente();
        Uri tarea = await CrearTareaAsync(cliente, "Pagar la luz");

        HttpResponseMessage respuesta = await cliente.DeleteAsync(tarea);

        Assert.Equal(HttpStatusCode.Unauthorized, respuesta.StatusCode);
    }

    [Fact]
    public async Task Delete_ConRolSinPermiso_Devuelve403()
    {
        HttpClient cliente = CrearCliente();
        Uri tarea = await CrearTareaAsync(cliente, "Pagar la luz");
        cliente.DefaultRequestHeaders.Add(AutenticacionDePrueba.EncabezadoRol, "Lector");

        HttpResponseMessage respuesta = await cliente.DeleteAsync(tarea);

        Assert.Equal(HttpStatusCode.Forbidden, respuesta.StatusCode);
    }

    [Fact]
    public async Task Delete_ComoAdmin_Devuelve204()
    {
        HttpClient cliente = CrearCliente();
        Uri tarea = await CrearTareaAsync(cliente, "Pagar la luz");
        cliente.DefaultRequestHeaders.Add(AutenticacionDePrueba.EncabezadoRol, "Admin");

        HttpResponseMessage respuesta = await cliente.DeleteAsync(tarea);

        Assert.Equal(HttpStatusCode.NoContent, respuesta.StatusCode);
    }

    [Fact]
    public async Task Vencidas_ConFechaFija_DevuelveSoloLasAnterioresAHoy()
    {
        var reloj = new FakeTimeProvider(new DateTimeOffset(2026, 3, 15, 12, 0, 0, TimeSpan.Zero));
        HttpClient cliente = CrearCliente(reloj);
        await CrearTareaAsync(cliente, "Vencida ayer", new DateOnly(2026, 3, 14));
        await CrearTareaAsync(cliente, "Vence hoy", new DateOnly(2026, 3, 15));
        await CrearTareaAsync(cliente, "Vence la semana que viene", new DateOnly(2026, 3, 22));

        var vencidas = await cliente.GetFromJsonAsync<List<TareaJson>>("/tareas/vencidas");

        TareaJson unica = Assert.Single(vencidas!);
        Assert.Equal("Vencida ayer", unica.Titulo);
    }

    [Fact]
    public async Task Vencidas_CuandoPasaElTiempo_IncluyeLasQueVencieron()
    {
        var reloj = new FakeTimeProvider(new DateTimeOffset(2026, 3, 15, 12, 0, 0, TimeSpan.Zero));
        HttpClient cliente = CrearCliente(reloj);
        await CrearTareaAsync(cliente, "Vence hoy", new DateOnly(2026, 3, 15));

        reloj.Advance(TimeSpan.FromDays(1));
        var vencidas = await cliente.GetFromJsonAsync<List<TareaJson>>("/tareas/vencidas");

        Assert.Equal("Vence hoy", Assert.Single(vencidas!).Titulo);
    }

    private sealed record TareaJson(int Id, string Titulo, DateOnly? Vence, bool Completada);
}
```

Ejecución con `dotnet test`. Resultado: **5 pruebas superadas, 0 con error** (más las de la lección 1, si están en el mismo proyecto).

Qué observar:

* Las tres pruebas de `DELETE` cubren los tres caminos de seguridad (`401`, `403`, `204`) sin un solo JWT ni secreto.
* El esquema JWT de la API sigue registrado, pero ya no es el predeterminado: `AddAuthentication("Test")` cambió el esquema por defecto solo en estas pruebas.
* La fecha es fija (15 de marzo de 2026, UTC): las pruebas dan el mismo resultado a cualquier hora.
* `reloj.Advance` simula el paso del tiempo **dentro** de una prueba: la misma tarea pasa de "no vencida" a "vencida".
* Cada `CrearCliente()` arranca su propia aplicación, así que el repositorio en memoria empieza vacío en cada prueba y `Assert.Single` es confiable.

-----

## Errores comunes

**1. Reemplazar servicios con `ConfigureServices` en lugar de `ConfigureTestServices`.**
Qué pasa: el servicio real sigue activo; el falso nunca se usa.
Por qué: `ConfigureServices` se aplica antes de `Program.cs`, y la API registra después el original, que gana.
Arreglo: `ConfigureTestServices`, que se aplica al final.

**2. Agregar un reemplazo de algo que se resuelve como colección.**
Qué pasa: corren el original y el falso (por ejemplo, dos handlers de notificación, y se envía un email real).
Por qué: `IEnumerable<T>` devuelve todos los registros, no solo el último.
Arreglo: `services.RemoveAll<T>()` antes de registrar el falso.

**3. Usar claves o tokens de producción en las pruebas.**
Qué pasa: secretos en el repositorio y pruebas que dependen de un entorno real.
Por qué: se intentó probar la autorización con tokens auténticos.
Arreglo: esquema de autenticación de prueba; si validas la configuración JWT, clave de prueba generada para eso.

**4. `DateTime.Now` en la API o en la prueba.**
Qué pasa: pruebas que fallan a ciertas horas o en otras zonas horarias.
Por qué: el reloj del sistema no es repetible.
Arreglo: `TimeProvider` inyectado en la API y `FakeTimeProvider` en la prueba.

**5. Un handler de prueba que siempre autentica.**
Qué pasa: los endpoints protegidos parecen funcionar, pero nunca se probó el `401`.
Por qué: el handler devuelve `Success` incluso sin datos de usuario.
Arreglo: `NoResult()` cuando la petición no trae la información de prueba, para poder probar el caso anónimo.

-----

## Según la versión de .NET

* **ASP.NET Core 2.1:** `ConfigureTestServices` y `WithWebHostBuilder`.
* **ASP.NET Core 7:** políticas de autorización en línea (`RequireAuthorization(p => ...)`) y esquema por defecto automático con un solo esquema registrado.
* **.NET 8:** `TimeProvider` en el framework y `FakeTimeProvider` en `Microsoft.Extensions.TimeProvider.Testing`; el constructor de `AuthenticationHandler` con `ISystemClock` queda obsoleto (se usa el de tres parámetros).

-----

## Cuándo sí y cuándo no

**Reemplaza un servicio cuando:** tiene efectos fuera del proceso (cobros, emails), depende de algo no determinista (reloj, aleatoriedad) o de un sistema externo que no controlas.

**Usa un esquema de autenticación de prueba cuando:** quieres probar autorización (roles, políticas, `401`/`403`) sin la complejidad de emitir y firmar tokens.

**No reemplaces cuando:** el servicio es tu propia lógica o el pipeline que quieres verificar; si reemplazas todo, ya no es una prueba de integración.

-----

## Resumen en 5 líneas

1. `ConfigureTestServices` registra después de la API: para servicios simples, gana el último.
2. Si el servicio se resuelve como colección, quita el original con `RemoveAll<T>()`.
3. `WithWebHostBuilder` crea una aplicación nueva por configuración, con estado propio.
4. Un handler de autenticación de prueba permite probar `401`, `403` y roles sin JWT ni secretos.
5. `TimeProvider` en la API y `FakeTimeProvider` en la prueba: el tiempo se vuelve repetible.

-----

## Para profundizar

<details>
<summary>Usuario de prueba más rico: claims a partir de varios encabezados</summary>

El handler puede leer más encabezados (`X-Test-Sub`, `X-Test-Scope`) para construir claims de identidad, scopes o permisos. Así pruebas políticas basadas en claims (por ejemplo, "solo el dueño de la tarea puede editarla") pasando distintos `sub` en cada petición. Mantenlo simple: si el handler crece demasiado, se vuelve una segunda implementación de autenticación que hay que mantener.

</details>

<details>
<summary>Servicios HTTP externos: falsear el <code>HttpMessageHandler</code></summary>

Si la API llama a otro servicio con `HttpClient` (registrado con `AddHttpClient`), puedes reemplazar su *handler* en lugar de la interfaz: `services.AddHttpClient<IClienteClima, ClienteClima>().ConfigurePrimaryHttpMessageHandler(() => new HandlerFalso(...))`. Así ejerces tu código de serialización y manejo de errores HTTP real, y solo la red queda simulada. Bibliotecas como WireMock.Net levantan un servidor HTTP falso completo para casos más elaborados.

</details>

-----

## En entrevista

### Respuesta corta (junior)

En las pruebas de integración reemplazo los servicios externos con `ConfigureTestServices`, por ejemplo una pasarela de pago falsa. Para endpoints protegidos uso un esquema de autenticación de prueba que crea el usuario con el rol que necesito, y para fechas, `FakeTimeProvider`.

### Respuesta ampliada (semi-senior)

Uso `ConfigureTestServices` porque corre después de `Program.cs` y el último registro gana; para servicios resueltos como colección uso `RemoveAll`. Cuando una prueba necesita configuración propia, `WithWebHostBuilder` crea una aplicación nueva con su estado. Para autorización registro un handler de prueba como esquema por defecto, que devuelve `NoResult` si no hay datos y así cubro `401`, `403` y el camino feliz sin tokens ni secretos; la validación JWT en sí la cubro aparte con una clave de prueba si hace falta. La API depende de `TimeProvider` y las pruebas usan `FakeTimeProvider`, con `Advance` para simular el paso del tiempo. Reemplazo solo lo externo o no determinista; el dominio y el pipeline van reales.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `ConfigureTestServices` y no `ConfigureServices`?**
Porque se aplica después de los registros de la API; `ConfigureServices` se aplicaría antes y sería sobrescrito.

**2. ¿Cómo pruebas un endpoint con `[Authorize(Roles = "Admin")]`?**
Con un esquema de autenticación de prueba que construye un usuario con el rol indicado en la petición.

**3. ¿Cómo haces repetible una prueba que depende de "hoy"?**
Inyectando `TimeProvider` en la API y reemplazándolo por `FakeTimeProvider` con una fecha fija.

**4. ¿Qué pierdes con `WithWebHostBuilder` por prueba?**
Velocidad: cada llamada arranca una aplicación nueva.

-----

## Práctica

**Ejercicio 1.** La API envía un email al crear una tarea, con `IEnviadorEmail`. Reemplázalo por un falso que guarde los emails y verifica que se envió uno con el título de la tarea.

<details>
<summary>Solución</summary>

```csharp
public sealed class EnviadorEmailFalso : IEnviadorEmail
{
    public List<(string Para, string Asunto)> Enviados { get; } = [];

    public Task EnviarAsync(string para, string asunto, CancellationToken ct)
    {
        lock (Enviados) Enviados.Add((para, asunto));
        return Task.CompletedTask;
    }
}

[Fact]
public async Task PostTarea_EnviaUnEmailConElTitulo()
{
    var emails = new EnviadorEmailFalso();
    HttpClient cliente = factory.WithWebHostBuilder(b => b.ConfigureTestServices(s =>
    {
        s.RemoveAll<IEnviadorEmail>();
        s.AddSingleton<IEnviadorEmail>(emails);
    })).CreateClient();

    await cliente.PostAsJsonAsync("/tareas", new { titulo = "Pagar la luz" });

    var (_, asunto) = Assert.Single(emails.Enviados);
    Assert.Contains("Pagar la luz", asunto);
}
```

La instancia falsa se registra como singleton para que la prueba conserve la referencia y pueda leer `Enviados` después de la petición. `RemoveAll` asegura que no quede registrado el enviador real.

</details>

**Ejercicio 2.** Haz que el handler de prueba lea también `X-Test-Sub` para el claim `sub` (con `"usuario-de-prueba"` por defecto), y escribe una prueba que llame a un endpoint `GET /me` que devuelve ese `sub`.

<details>
<summary>Solución</summary>

En el handler:

```csharp
string sub = Request.Headers.TryGetValue("X-Test-Sub", out var valor) ? valor.ToString() : "usuario-de-prueba";
var identidad = new ClaimsIdentity(
    [new Claim("sub", sub), new Claim(ClaimTypes.Role, rol.ToString())],
    Esquema);
```

En la API: `app.MapGet("/me", (ClaimsPrincipal u) => TypedResults.Ok(new { Sub = u.FindFirstValue("sub") })).RequireAuthorization();`

Prueba:

```csharp
[Fact]
public async Task Me_DevuelveElSubDelUsuarioAutenticado()
{
    HttpClient cliente = CrearCliente();
    cliente.DefaultRequestHeaders.Add(AutenticacionDePrueba.EncabezadoRol, "Lector");
    cliente.DefaultRequestHeaders.Add("X-Test-Sub", "ana");

    var yo = await cliente.GetFromJsonAsync<MeJson>("/me");

    Assert.Equal("ana", yo!.Sub);
}

private sealed record MeJson(string Sub);
```

Así puedes probar reglas que dependen de **quién** es el usuario (por ejemplo, que solo el dueño edite su tarea) cambiando un encabezado.

</details>

-----

## Siguiente lección

[Base de datos real con Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md)
