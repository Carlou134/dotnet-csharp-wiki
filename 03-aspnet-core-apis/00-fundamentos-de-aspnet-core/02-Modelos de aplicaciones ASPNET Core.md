# Modelos de aplicaciones ASP.NET Core

## En una frase

Minimal APIs, controladores, MVC con vistas, Razor Pages y Blazor comparten el mismo host, DI y middleware de ASP.NET Core, pero organizan el código y la interfaz de formas distintas; elegir bien depende de qué entregas (JSON, HTML de servidor o UI interactiva), no de cuál parece "más profesional".

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo se registran servicios y se arma el pipeline: [Host, configuración y pipeline HTTP](01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* Atributos y records: [Sintaxis moderna de C#](../../01-csharp-core-and-runtime/11-codigo-limpio/03-Sintaxis%20moderna%20de%20CSharp.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Minimal API:** endpoints declarados con `app.MapGet`/`MapPost` y un handler.
* **Controlador:** clase que agrupa acciones HTTP relacionadas.
* **MVC:** patrón que separa modelo, vista y controlador.
* **Razor Pages:** modelo de UI centrado en páginas, cada una con su `PageModel`.
* **Blazor:** modelo de UI basado en componentes Razor.
* **Modo de renderizado:** dónde y cómo se ejecuta un componente Blazor (servidor, navegador o solo HTML estático).

-----

## El problema

Un equipo crea una API de pedidos copiando un controlador de un proyecto MVC con vistas:

```csharp
public sealed class PedidosController(IPedidoRepository repo) : Controller
{
    [HttpPost("api/pedidos")]
    public IActionResult Crear(CrearPedido pedido)
    {
        repo.Guardar(pedido);
        return Ok();
    }
}

public sealed record CrearPedido([Required] string Cliente, [Range(1, 100)] int Cantidad);
```

El cliente envía JSON:

```http
POST /api/pedidos
Content-Type: application/json

{ "cliente": "Ana", "cantidad": 0 }
```

La respuesta es `200 OK`. Se guarda un pedido con `Cliente = null` y `Cantidad = 0`. ¿Por qué?

* Sin `[ApiController]`, MVC **no lee el cuerpo JSON** para un tipo complejo: lo busca en el formulario y en la query string. No lo encuentra y crea el record con valores por defecto.
* Sin `[ApiController]`, un `ModelState` inválido **no detiene** la acción. Los atributos `[Required]` y `[Range]` detectaron el error, pero nadie lo miró.

El modelo de aplicación no es un detalle estético. Cada uno trae convenciones distintas de binding, validación y respuesta.

Las consecuencias de elegir sin entender:

* datos inválidos persistidos, como en el ejemplo;
* ceremonias (vistas, antiforgery, layouts) en un servicio que solo devuelve JSON;
* comparar MAUI con una Web API como si fueran alternativas, cuando una es el cliente y la otra el servidor.

-----

## Cómo funciona

### 1. Todos comparten la base

```text
                  ┌──────────────── ASP.NET Core ────────────────┐
                  │ host · DI · configuración · logging · Kestrel │
                  │ middleware · routing · auth                   │
                  └───────────────────────┬───────────────────────┘
       ┌──────────────┬───────────────┬───┴──────────┬──────────────┐
  Minimal API   API con           MVC con         Razor Pages      Blazor
  (MapGet...)   controladores     vistas          (PageModel)      (componentes)
     JSON          JSON           HTML servidor   HTML servidor    UI interactiva
```

Lo que aprendiste del pipeline aplica a todos. Cambia lo que ocurre **después** del routing.

### 2. Comparación

| Modelo | Unidad principal | Entrega | Uso típico |
| --- | --- | --- | --- |
| Minimal API | Handler (`MapGet`, `MapPost`) | JSON | APIs, microservicios, *vertical slices* |
| API con controladores | Controlador `: ControllerBase` + acciones | JSON | APIs grandes, equipos acostumbrados a MVC, filtros MVC |
| MVC con vistas | Controlador `: Controller` + vista `.cshtml` | HTML | Sitios con muchas pantallas renderizadas en servidor |
| Razor Pages | Página `.cshtml` + `PageModel` | HTML | Formularios y flujos centrados en una página |
| Blazor Web App | Componente `.razor` | HTML + interactividad | UI rica escrita en C# |

### 3. API con controladores: `ControllerBase` y `[ApiController]`

```csharp
[ApiController]
[Route("api/pedidos")]
public sealed class PedidosController : ControllerBase
```

* `ControllerBase` trae lo necesario para APIs (`Ok`, `NotFound`, `CreatedAtAction`). `Controller` le agrega soporte de vistas, que una API no necesita.
* `[ApiController]` activa convenciones de API: los tipos complejos se leen del cuerpo, un `ModelState` inválido responde `400` con ProblemDetails automáticamente, y el routing por atributos es obligatorio.

### 4. Minimal API

```csharp
app.MapPost("/api/pedidos", (CrearPedido pedido) => TypedResults.Created("/api/pedidos/1", pedido));
```

Menos estructura y menos convenciones implícitas. En .NET 10 la validación de DataAnnotations se activa con `builder.Services.AddValidation()`. "Minimal" describe la forma de declarar endpoints, **no el tamaño máximo de la aplicación**: con `MapGroup` y endpoints en clases estáticas escala bien.

### 5. MVC con vistas y Razor Pages

Las dos renderizan HTML en el servidor con sintaxis Razor.

* **MVC** separa: el controlador recibe la petición y elige una vista; la vista presenta un modelo. Bueno cuando muchas acciones comparten un controlador.
* **Razor Pages** agrupa: la página y su `PageModel` (con `OnGet`, `OnPost`) viven juntos. Bueno para formularios, donde cada pantalla es una unidad.

Razor es la **sintaxis**; Razor Pages es un **modelo de aplicación** que la usa. MVC y Blazor también usan Razor.

### 6. Blazor y sus modos de renderizado

| Modo | Dónde corre el componente | Costo |
| --- | --- | --- |
| SSR estático | Servidor, una sola vez por petición | Sin interactividad propia |
| Interactive Server | Servidor, con eventos por una conexión SignalR | Una conexión y estado por usuario |
| Interactive WebAssembly | Navegador, con el runtime .NET descargado | Descarga inicial grande |
| Interactive Auto | Server al principio, WebAssembly cuando ya se descargó | Complejidad de ambos |

Blazor no significa "cero JavaScript": el navegador sigue siendo la plataforma, y muchas integraciones necesitan *JS interop*.

### 7. MAUI no es un modelo de ASP.NET Core

.NET MAUI crea aplicaciones nativas de escritorio y móviles. No sirve HTTP: **consume** una API. La pregunta "¿MAUI o Web API?" está mal planteada; lo habitual es MAUI como cliente y ASP.NET Core como servidor.

### 8. Cómo elegir

```text
¿Qué entregas?
├── Datos (JSON) para otros sistemas o un front aparte
│     ├── API chica/mediana o preferencia por menos ceremonia ──► Minimal API
│     └── API grande, filtros MVC, equipo con experiencia MVC ──► Controladores + [ApiController]
├── HTML renderizado en servidor
│     ├── Pantallas centradas en formularios ──► Razor Pages
│     └── Muchas acciones por recurso ──► MVC con vistas
└── UI interactiva escrita en C# ──► Blazor (elige el modo de renderizado según latencia y escala)
```

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`. Una minimal API y un controlador conviven en la misma aplicación:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

var app = builder.Build();

app.MapGet("/api/status", () => TypedResults.Ok(new StatusResponse("ready")));
app.MapControllers();

app.Run();

public sealed record StatusResponse(string Status);

public sealed record CrearPedido([Required] string Cliente, [Range(1, 100)] int Cantidad);

[ApiController]
[Route("api/pedidos")]
public sealed class PedidosController : ControllerBase
{
    [HttpPost]
    public IActionResult Crear(CrearPedido pedido) => Created("/api/pedidos/1", pedido);
}
```

Peticiones:

```text
GET  /api/status                                   → 200 {"status":"ready"}

POST /api/pedidos  {"cliente":"Ana","cantidad":2}  → 201 {"cliente":"Ana","cantidad":2}
                                                     Location: /api/pedidos/1

POST /api/pedidos  {"cliente":"Ana","cantidad":0}  → 400 application/problem+json
                                                     errors.Cantidad: ["The field Cantidad must be between 1 and 100."]
```

Qué observar:

* `[ApiController]` leyó el JSON del cuerpo y respondió `400` sin que la acción se ejecutara. Es exactamente lo que faltaba en el problema.
* Los atributos están en los **parámetros** del record posicional: así los lee MVC.
* La minimal API y el controlador comparten el mismo host y el mismo pipeline.

-----

## Errores comunes

**1. Usar `Controller` sin `[ApiController]` para una API.**
Qué pasa: el JSON no se enlaza y los datos inválidos llegan a la acción, como en el problema.
Por qué: las convenciones de API solo se activan con el atributo.
Arreglo: `ControllerBase` + `[ApiController]` + `[Route]`.

**2. Decir que MVC es la arquitectura de la aplicación.**
Qué pasa: los controladores acumulan reglas de negocio, consultas y cálculos.
Por qué: MVC organiza la **presentación**; no dice dónde viven el dominio ni la persistencia.
Arreglo: el controlador delega en casos de uso; el dominio no depende de MVC.

**3. Elegir Blazor para "no aprender web".**
Qué pasa: problemas de latencia, estado o accesibilidad que no se entienden.
Por qué: Blazor abstrae parte del trabajo, pero sigue corriendo sobre HTTP, HTML, CSS y el navegador.
Arreglo: aprende los fundamentos web y elige el modo de renderizado a conciencia.

**4. Confundir Razor con Razor Pages.**
Qué pasa: se dice "uso Razor" sin aclarar si son vistas MVC, páginas o componentes.
Por qué: Razor es la sintaxis; Razor Pages es un modelo de aplicación.
Arreglo: nombra el modelo concreto.

**5. Mezclar modelos sin un límite claro.**
Qué pasa: la mitad de los endpoints son minimal y la otra mitad controladores, sin criterio; los filtros y las políticas se configuran dos veces.
Por qué: los modelos pueden convivir, pero cada uno tiene sus propias extensiones.
Arreglo: mezcla solo con un límite explícito (por ejemplo, API en minimal y back-office en Razor Pages) y documéntalo.

-----

## Según la versión de .NET

* **ASP.NET Core 1.0:** unifica MVC y Web API en un mismo stack (en .NET Framework eran dos frameworks distintos).
* **ASP.NET Core 2.0:** incorpora Razor Pages.
* **ASP.NET Core 2.1:** incorpora `[ApiController]` y `ControllerBase` para APIs.
* **ASP.NET Core 3.0:** Blazor Server.
* **ASP.NET Core 3.2 / .NET 5:** Blazor WebAssembly.
* **.NET 6:** minimal APIs y hosting mínimo.
* **.NET 8:** Blazor Web App unifica los modos de renderizado en una sola plantilla.
* **.NET 10:** las minimal APIs validan DataAnnotations con `AddValidation()`.

-----

## Cuándo sí y cuándo no

**Usa minimal APIs cuando:** expones JSON y quieres poca ceremonia. **No las uses cuando:** dependes de filtros de MVC existentes o de convenciones que el equipo ya domina en controladores.

**Usa MVC o Razor Pages cuando:** el servidor entrega HTML y la interactividad es moderada. **No los uses cuando:** solo devuelves JSON.

**Usa Blazor cuando:** necesitas UI interactiva y quieres compartir C# entre cliente y servidor. **No lo uses cuando:** la página es casi estática o tu equipo ya tiene un front en otro framework.

-----

## Resumen en 5 líneas

1. Todos los modelos comparten host, DI, configuración y middleware.
2. Para APIs con controladores: `ControllerBase` + `[ApiController]`, que activa binding del cuerpo y `400` automático.
3. MVC y Razor Pages renderizan HTML en el servidor; Razor es solo la sintaxis.
4. Blazor organiza UI en componentes y se ejecuta según su modo de renderizado.
5. MAUI es un cliente nativo que consume una API; no es un modelo de ASP.NET Core.

-----

## Para profundizar

<details>
<summary>Qué hace exactamente [ApiController]</summary>

* Exige routing por atributos.
* Infiere el origen de los parámetros: tipos complejos desde el cuerpo, `IFormFile` desde el formulario, parámetros de ruta desde la ruta.
* Responde `400` con `ValidationProblemDetails` si `ModelState` es inválido, antes de ejecutar la acción.
* Convierte respuestas de error (`NotFound()`, `BadRequest()`) en ProblemDetails.

Puedes aplicarlo a nivel de ensamblado con `[assembly: ApiController]` para no repetirlo.

</details>

<details>
<summary>Endpoint routing: por qué pueden convivir</summary>

Desde ASP.NET Core 3.0, minimal APIs, controladores, Razor Pages y componentes Blazor se registran como **endpoints** en la misma tabla de routing. Por eso un mismo pipeline sirve a todos, y la autorización funciona igual para un `MapGet` que para una acción de controlador.

</details>

-----

## En entrevista

### Respuesta corta (junior)

ASP.NET Core permite APIs (minimal o con controladores), MVC con vistas, Razor Pages y Blazor. La elección depende de si entregas datos JSON, HTML renderizado en el servidor o UI interactiva. Todos comparten el mismo host y middleware.

### Respuesta ampliada (semi-senior)

Los modelos comparten host, DI, configuración y middleware; cambia la unidad de organización y lo que pasa después del routing. Para APIs uso minimal APIs o controladores con `[ApiController]`, que trae binding del cuerpo y validación automática. Para HTML de servidor, Razor Pages en flujos de formularios o MVC si hay muchas acciones por recurso. Blazor lo elijo por su modo de renderizado, sabiendo el costo de un circuito SignalR por usuario o de la descarga de WebAssembly. MVC es presentación, no la arquitectura del negocio.

### Preguntas frecuentes de seguimiento

**1. ¿Minimal API significa aplicación pequeña?**
No. Describe cómo se declaran los endpoints; con `MapGroup` escala bien.

**2. ¿Qué cambia `[ApiController]`?**
Binding del cuerpo, `400` automático, routing por atributos y errores como ProblemDetails.

**3. ¿MAUI reemplaza ASP.NET Core?**
No. MAUI es un cliente; puede consumir una API ASP.NET Core.

**4. ¿Puedo mezclar minimal APIs y controladores?**
Sí, comparten endpoint routing. Conviene hacerlo con un límite claro.

-----

## Práctica

**Ejercicio 1.** Elige un modelo para cada caso y justifica: (a) panel administrativo interno con formularios y poco JavaScript; (b) API para una app móvil; (c) tablero que actualiza gráficos cada segundo, para 20 usuarios internos.

<details>
<summary>Solución</summary>

* (a) Razor Pages: cada pantalla es un formulario con su `PageModel`. MVC también sirve si hay muchas acciones por recurso.
* (b) Minimal API o controladores con `[ApiController]`: el cliente es MAUI u otra app; el servidor solo entrega JSON.
* (c) Blazor Interactive Server: 20 usuarios internos con baja latencia de red hacen aceptable un circuito por usuario, y no hace falta descargar WebAssembly.

</details>

**Ejercicio 2.** Corrige el controlador del problema para que rechace `{"cliente":"Ana","cantidad":0}` con `400` sin escribir ningún `if`.

<details>
<summary>Solución</summary>

```csharp
[ApiController]
[Route("api/pedidos")]
public sealed class PedidosController(IPedidoRepository repo) : ControllerBase
{
    [HttpPost]
    public IActionResult Crear(CrearPedido pedido)
    {
        repo.Guardar(pedido);
        return Created("/api/pedidos/1", pedido);
    }
}
```

`ControllerBase` en lugar de `Controller`, `[ApiController]` para leer el cuerpo y validar, y `[Route]` porque el atributo exige routing por atributos. Con `cantidad: 0`, el filtro automático responde `400` y `repo.Guardar` nunca se ejecuta.

</details>

-----

## Siguiente lección

[MVC, routing y vistas Razor](03-MVC%20routing%20y%20vistas%20Razor.md)
