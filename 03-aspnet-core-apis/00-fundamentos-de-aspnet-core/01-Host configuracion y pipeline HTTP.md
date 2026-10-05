# Host, configuración y pipeline HTTP

## En una frase

Una aplicación ASP.NET Core primero **registra** servicios y fuentes de configuración, después **construye** el host, y a partir de ahí procesa cada petición atravesando un pipeline ordenado de middleware hasta llegar a un endpoint.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es el SDK, el `.csproj` y `dotnet run`: [SDK, CLI, proyectos y NuGet](../../01-csharp-core-and-runtime/00-introduccion/07-SDK%20CLI%20proyectos%20y%20NuGet.md).
* Interfaces y clases: [POO](../../01-csharp-core-and-runtime/04-poo/README.md).
* `async`/`await`: [Asincronía](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Host:** objeto que reúne configuración, logging, inyección de dependencias, servidor web y ciclo de vida.
* **Inyección de dependencias (DI):** el contenedor crea los objetos que un componente necesita y se los entrega.
* **Middleware:** componente del pipeline que procesa la petición antes y después del siguiente.
* **Endpoint:** destino final que el routing selecciona para una petición.
* **Options pattern:** forma de leer una sección de configuración como una clase tipada.

-----

## El problema

Este `Program.cs` se armó copiando fragmentos de distintos tutoriales:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();

builder.Services.AddSingleton<PedidoService>();   // después de Build

app.UseAuthorization();                            // antes de autenticar
app.UseAuthentication();

app.MapGet("/pedidos", (PedidoService s) => s.Listar()).RequireAuthorization();

app.Run("http://localhost:5000");                  // puerto fijo
```

Qué pasa al ejecutarlo:

* La línea `AddSingleton` después de `Build` lanza `InvalidOperationException: The service collection cannot be modified because it is read-only.` La aplicación ni siquiera arranca.
* Si se corrige eso, `UseAuthorization` corre antes de que exista un usuario autenticado: **toda** petición a `/pedidos` recibe `401`, aunque traiga un token válido.
* `app.Run("http://localhost:5000")` ignora la configuración de puertos. En un contenedor que espera el `8080`, la API escucha en otro puerto y en `localhost`, inalcanzable desde afuera.

Ninguno de estos errores está en la lógica de negocio. Están en **cuándo** se registra algo y en **qué orden** se ejecuta.

-----

## Cómo funciona

### 1. Dos fases: registrar y ejecutar

```text
WebApplication.CreateBuilder(args)
   │  configuración, logging, DI, Kestrel (servidor web)
   ▼
builder.Services.Add...      ◄── FASE DE REGISTRO: describes qué existe
builder.Configuration...
   │
builder.Build()              ◄── se congela la colección de servicios
   │
   ▼
app.Use... / app.Map...      ◄── FASE DE PIPELINE: describes por dónde pasa cada petición
app.Run()                    ◄── arranca el servidor y espera peticiones
```

Después de `Build()`, los servicios son de solo lectura. Por eso el `AddSingleton` del problema falla.

### 2. Inyección de dependencias y ciclos de vida

```csharp
builder.Services.AddSingleton<ICatalogo, CatalogoEnMemoria>();
builder.Services.AddScoped<IPedidoRepository, PedidoRepository>();
builder.Services.AddTransient<ICalculadoraEnvio, CalculadoraEnvio>();
```

| Lifetime | Una instancia por... | Uso típico |
| --- | --- | --- |
| Singleton | Toda la aplicación | Cachés, `TimeProvider`, clientes sin estado |
| Scoped | Cada petición HTTP | `DbContext`, unidad de trabajo |
| Transient | Cada vez que se pide | Servicios livianos sin estado |

Registrar no ejecuta nada: describe cómo crear el objeto cuando un endpoint lo pida.

La regla que no se rompe: **un singleton no puede depender de un scoped**. Viviría para siempre con una instancia que debía morir al final de la petición (una *captive dependency*). En Development, ASP.NET Core lo detecta al resolver:

```text
InvalidOperationException: Cannot consume scoped service 'IPedidoRepository' from singleton 'ICatalogo'.
```

### 3. Configuración por capas

`CreateBuilder` carga estas fuentes, en este orden. **La posterior sobrescribe a la anterior:**

```text
1. appsettings.json
2. appsettings.{Environment}.json     (por ejemplo, appsettings.Development.json)
3. User secrets                       (solo en Development)
4. Variables de entorno               Store__Name=Arequipa
5. Argumentos de línea de comandos    --Store:Name=Cusco
```

Las secciones anidadas se escriben con `:` en código y argumentos, y con `__` (doble guion bajo) en variables de entorno, porque `:` no es válido en todos los sistemas.

Esa precedencia permite que el mismo artefacto se configure distinto en cada entorno, sin recompilar.

### 4. Options pattern: configuración tipada y validada

Leer `builder.Configuration["Store:Name"]` por todas partes dispersa strings mágicos y no valida nada. Mejor:

```csharp
builder.Services.AddOptions<StoreOptions>()
    .BindConfiguration("Store")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Con `ValidateOnStart`, una configuración inválida **impide arrancar**. Es mejor que descubrirlo con la primera petición en producción.

### 5. Entornos

El entorno sale de la variable `ASPNETCORE_ENVIRONMENT` (o `DOTNET_ENVIRONMENT`). **Si no está definida, el entorno es `Production`.**

`Development`, `Staging` y `Production` cambian comportamiento operativo: páginas de error detalladas, user secrets, validación de scopes. No son una frontera de seguridad: un secreto sigue sin pertenecer al repositorio, en ningún entorno.

### 6. El pipeline: orden de cebolla

```text
petición ──►
  UseExceptionHandler   atrapa excepciones de todo lo que está adentro
    UseHsts / UseHttpsRedirection
      UseStaticFiles     puede responder y cortar (short-circuit)
        UseRouting        elige el endpoint
          UseCors
            UseAuthentication   construye HttpContext.User
              UseAuthorization    lee la policy del endpoint elegido
                endpoint            tu handler
◄── respuesta (recorre las capas en orden inverso)
```

* El manejador de excepciones va primero para envolver a todos los demás.
* Los archivos estáticos van antes del routing: no necesitan autenticación ni endpoint.
* **Routing va antes de autorización** porque la autorización necesita saber qué endpoint se eligió para leer su `RequireAuthorization` o `[Authorize]`.
* **Autenticación va antes de autorización** porque no puedes decidir permisos sin saber quién es el usuario.

En minimal APIs, `WebApplication` agrega `UseRouting`, `UseAuthentication` y `UseAuthorization` automáticamente si no los llamas. Llámalos explícitamente cuando necesites ubicar otro middleware (como CORS) entre ellos.

### 7. `launchSettings.json` y los puertos

`Properties/launchSettings.json` define perfiles para `dotnet run` y el IDE: URLs locales y `ASPNETCORE_ENVIRONMENT=Development`. **No se publica.** En un servidor o contenedor, los puertos vienen de `ASPNETCORE_URLS`, `ASPNETCORE_HTTP_PORTS` o la configuración de Kestrel. Las imágenes de contenedor de .NET 8 o posterior escuchan en el `8080`.

### 8. `wwwroot`

Es la raíz de los archivos estáticos. Todo lo que pongas ahí es **público**: el middleware de archivos estáticos lo sirve sin autenticación.

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOptions<StoreOptions>()
    .BindConfiguration(StoreOptions.Section)
    .ValidateDataAnnotations()
    .ValidateOnStart();
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}
app.UseHttpsRedirection();

app.Use(async (context, next) =>
{
    context.Response.Headers["X-App"] = "Orders";
    await next(context);
});

app.MapGet("/info", (IOptions<StoreOptions> options, IHostEnvironment env) =>
    TypedResults.Ok(new { Store = options.Value.Name, Environment = env.EnvironmentName }));

app.MapGet("/time", (TimeProvider clock) => TypedResults.Ok(clock.GetUtcNow()));

app.Run();

public sealed class StoreOptions
{
    public const string Section = "Store";

    [Required]
    public string Name { get; set; } = "";
}
```

En `appsettings.json`, agrega la sección:

```json
{
  "Store": { "Name": "Lima" }
}
```

Ejecuta con `dotnet run --launch-profile https` y prueba:

```text
GET /info  → 200 {"store":"Lima","environment":"Development"}
             header X-App: Orders
```

Ahora define una variable de entorno antes de ejecutar (en PowerShell: `$env:Store__Name = "Arequipa"`):

```text
GET /info  → 200 {"store":"Arequipa","environment":"Development"}
```

Por último, borra la sección `Store` de `appsettings.json` y quita la variable. La aplicación **no arranca**:

```text
Microsoft.Extensions.Options.OptionsValidationException: DataAnnotation validation failed for 'StoreOptions' members: 'Name' with the error: 'The Name field is required.'.
```

Qué observar:

* La variable de entorno ganó sobre `appsettings.json` sin recompilar: así se configura un mismo artefacto en distintos entornos.
* `ValidateOnStart` convirtió una configuración faltante en un fallo inmediato y explícito.
* `/time` devuelve la hora real de la ejecución, por eso no se muestra su salida. `TimeProvider` permite reemplazar el reloj en las pruebas.
* El header `X-App` aparece en todas las respuestas porque el middleware lo agrega antes de llamar a `next`.

-----

## Errores comunes

**1. Registrar servicios después de `Build()`.**
Qué pasa: `InvalidOperationException: The service collection cannot be modified because it is read-only.`
Por qué: `Build()` congela el contenedor.
Arreglo: todo `builder.Services.Add...` va antes de `builder.Build()`.

**2. Autorizar antes de autenticar.**
Qué pasa: `401` en endpoints protegidos aunque el token sea válido.
Por qué: cuando corre la autorización, `HttpContext.User` todavía está vacío.
Arreglo: `UseRouting` → `UseAuthentication` → `UseAuthorization`, o deja que `WebApplication` los agregue.

**3. Un singleton que depende de un scoped.**
Qué pasa: `InvalidOperationException: Cannot consume scoped service ... from singleton ...` en Development; en Production, un `DbContext` compartido entre peticiones.
Por qué: el singleton captura una instancia que debía vivir solo una petición.
Arreglo: baja el lifetime del consumidor a scoped, o crea un scope con `IServiceScopeFactory` cuando realmente lo necesites.

**4. Guardar secretos en `appsettings.json`.**
Qué pasa: la contraseña queda en el repositorio y en cada artefacto publicado.
Por qué: `appsettings.json` es configuración versionada, no un almacén de secretos.
Arreglo: user secrets en desarrollo; variables de entorno o un gestor de secretos en producción.

**5. Fijar puertos en el código o confiar en `launchSettings.json`.**
Qué pasa: en el servidor o el contenedor la API escucha en otro puerto, o solo en `localhost`.
Por qué: `launchSettings.json` no se publica, y `app.Run(url)` pisa la configuración.
Arreglo: `app.Run()` sin argumentos y puertos desde `ASPNETCORE_HTTP_PORTS` o la configuración del entorno.

-----

## Según la versión de .NET

* **ASP.NET Core 1.x y 2.x:** arranque repartido entre `Program.cs` y `Startup.cs` (`ConfigureServices` y `Configure`).
* **ASP.NET Core 3.0:** host genérico y *endpoint routing*.
* **.NET 6:** modelo de hosting mínimo con `WebApplication`; todo en `Program.cs`.
* **.NET 7:** `WebApplication` agrega automáticamente autenticación y autorización si registraste sus servicios.
* **.NET 8:** `ASPNETCORE_HTTP_PORTS`, puerto `8080` en contenedores y `IExceptionHandler`.
* **.NET 9:** las plantillas reemplazan `UseStaticFiles` por `MapStaticAssets`, que agrega compresión y *fingerprinting*.
* **.NET 10:** el modelo de `WebApplication` se mantiene; `Startup.cs` sigue siendo válido, pero no se usa en material nuevo.

-----

## Cuándo sí y cuándo no

**Crea un middleware cuando:** una política HTTP debe envolver a muchas o todas las rutas (headers comunes, correlación, manejo de errores).

**No lo uses cuando:** la lógica pertenece a un único endpoint (usa un *endpoint filter*) o al negocio (va en el dominio o en un caso de uso).

**Usa el options pattern cuando:** una sección de configuración se lee en más de un lugar o debe validarse. **No hace falta cuando:** lees un único valor una sola vez al arrancar.

-----

## Resumen en 5 líneas

1. Primero se registran servicios y configuración; `Build()` congela el contenedor.
2. Singleton, scoped y transient definen cuánto vive cada instancia; un singleton no depende de un scoped.
3. La configuración se apila por capas y la última fuente gana, sin recompilar.
4. Los middleware se ejecutan en orden de cebolla: routing → autenticación → autorización → endpoint.
5. `launchSettings.json` es local: los puertos y secretos de producción vienen del entorno.

-----

## Para profundizar

<details>
<summary>Short-circuit: cortar el pipeline</summary>

Un middleware puede responder sin llamar a `next`. `UseStaticFiles` lo hace cuando encuentra el archivo: la petición nunca llega al routing. Es eficiente, pero peligroso si se coloca mal: un middleware que corta antes de `UseAuthorization` puede servir contenido sin verificar permisos.

</details>

<details>
<summary>Middleware como clase</summary>

Para lógica reutilizable o con dependencias, escribe una clase con un método `InvokeAsync(HttpContext context)` y regístrala con `app.UseMiddleware<T>()`. El middleware convencional se crea **una sola vez** (es un singleton de hecho): los servicios scoped se reciben como parámetros de `InvokeAsync`, no en el constructor.

</details>

-----

## En entrevista

### Respuesta corta (junior)

ASP.NET Core registra servicios en un contenedor de DI, construye la aplicación y procesa cada petición mediante middleware ordenado hasta llegar a un endpoint. La configuración se combina desde `appsettings.json`, variables de entorno y otras fuentes, y la última gana.

### Respuesta ampliada (semi-senior)

El host integra configuración, DI, logging, Kestrel y ciclo de vida. Elijo lifetimes con cuidado: `DbContext` scoped, nunca capturado por un singleton. Uso el options pattern con `ValidateOnStart` para fallar al arrancar si falta configuración. El pipeline es de cebolla: manejo de excepciones afuera, estáticos antes del routing, y routing, autenticación y autorización en ese orden. Los secretos y puertos vienen del entorno, no de `appsettings.json` ni de `launchSettings.json`.

### Preguntas frecuentes de seguimiento

**1. ¿Una carpeta `Controllers` registra controladores?**
No. Hay que llamar a `AddControllers` y `MapControllers`; las carpetas son convención, no configuración.

**2. ¿El puerto siempre es 5000?**
No. Depende del perfil local, de `ASPNETCORE_URLS`/`ASPNETCORE_HTTP_PORTS` o de la plataforma. En contenedores .NET 8+ es `8080`.

**3. ¿Qué entorno usa la app si no defines ninguno?**
`Production`.

**4. ¿Por qué el manejador de excepciones va primero?**
Porque solo atrapa excepciones de los middleware que están adentro de él.

-----

## Práctica

**Ejercicio 1.** Agrega un middleware que mida cuánto tarda cada petición y escriba el resultado en el log con `ILogger`. ¿Dónde lo ubicarías en el pipeline?

<details>
<summary>Solución</summary>

```csharp
app.Use(async (context, next) =>
{
    var inicio = Stopwatch.GetTimestamp();
    try
    {
        await next(context);
    }
    finally
    {
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogInformation("{Metodo} {Ruta} → {Status} en {Ms:N1} ms",
            context.Request.Method, context.Request.Path,
            context.Response.StatusCode, Stopwatch.GetElapsedTime(inicio).TotalMilliseconds);
    }
});
```

Agrega `using System.Diagnostics;`. Va justo después de `UseExceptionHandler`, para medir todo lo demás. El `finally` registra la duración aunque haya una excepción. En producción no hace falta escribirlo: ASP.NET Core ya publica la métrica `http.server.request.duration`.

</details>

**Ejercicio 2.** Este registro falla en Development al pedir `/reportes`. ¿Por qué y cómo lo corriges?

```csharp
builder.Services.AddDbContext<AppDbContext>(...);         // scoped
builder.Services.AddSingleton<GeneradorReportes>();       // recibe AppDbContext en el constructor
```

<details>
<summary>Solución</summary>

`GeneradorReportes` es singleton y depende de `AppDbContext`, que es scoped: `Cannot consume scoped service 'AppDbContext' from singleton 'GeneradorReportes'`. Si no se detectara, todas las peticiones compartirían el mismo `DbContext`, que no es seguro entre hilos. Arreglo: `AddScoped<GeneradorReportes>()`.

</details>

-----

## Siguiente lección

[Modelos de aplicaciones ASP.NET Core](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md)
