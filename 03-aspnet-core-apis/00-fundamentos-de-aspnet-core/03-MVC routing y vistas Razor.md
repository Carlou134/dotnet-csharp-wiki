# MVC, routing y vistas Razor

## En una frase

En MVC, el routing elige un controlador y una acción, la acción coordina el caso de uso y entrega un ViewModel, y una vista Razor lo convierte en HTML; cada pieza tiene una responsabilidad y las convenciones de nombres las conectan.

-----

## Antes de empezar

Conviene que ya sepas:

* Cuándo elegir MVC frente a otros modelos: [Modelos de aplicaciones ASP.NET Core](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md).
* Inyección de dependencias y pipeline: [Host, configuración y pipeline HTTP](01-Host%20configuracion%20y%20pipeline%20HTTP.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Action:** método público de un controlador que el routing puede invocar.
* **Ruta convencional:** patrón global que traduce URLs a controlador y acción.
* **Model binding:** construcción de los parámetros de una acción a partir de la petición HTTP.
* **ViewModel:** clase o record diseñado para lo que necesita una vista.
* **Tag Helper:** atributo de servidor (como `asp-action`) que genera HTML a partir del modelo y las rutas.

-----

## El problema

Un controlador de la primera versión del sitio:

```csharp
public class HomeController(AppDbContext db) : Controller
{
    public IActionResult Index()
    {
        var pedidos = db.Pedidos.Include(p => p.Cliente).ToList();
        foreach (var p in pedidos)
            p.Total = p.Total * 0.9m;              // descuento "temporal"
        ViewBag.Html = "<b>" + pedidos.First().Cliente.Nombre + "</b>";
        return View(pedidos);
    }
}
```

Y su vista hace `@Html.Raw(ViewBag.Html)`.

Funciona al principio. Después:

* Alguien renombra `HomeController` a `PedidosController` para que la URL sea más clara. La página falla:

  ```text
  InvalidOperationException: The view 'Index' was not found. The following locations were searched:
  /Views/Pedidos/Index.cshtml
  /Views/Shared/Index.cshtml
  ```

* El descuento "temporal" se aplica a las **entidades rastreadas**: si otra parte del código llama a `SaveChanges`, los totales reales cambian en la base.
* Un cliente se registra con el nombre `<script>...</script>`. `Html.Raw` lo inserta tal cual: es un ataque XSS.
* La vista recibe la entidad completa, con datos que no muestra pero que quedan acoplados al esquema.

El controlador mezcla transporte, negocio, persistencia y presentación. Y las convenciones de MVC, que nadie explicó, rompen la página con un simple renombrado.

-----

## Cómo funciona

### 1. El recorrido de una petición

```text
GET /Pedidos/Detalle/42
        │
        ▼ routing: patrón "{controller=Pedidos}/{action=Index}/{id?}"
controller = "Pedidos" → PedidosController
action     = "Detalle" → método Detalle
id         = "42"      → model binding lo convierte a int
        │
        ▼
PedidosController.Detalle(42)
   ├── pide los datos a un servicio o caso de uso
   ├── arma un ViewModel
   └── return View(viewModel)
        │
        ▼ búsqueda de vista: /Views/Pedidos/Detalle.cshtml, luego /Views/Shared/Detalle.cshtml
Razor renderiza HTML ──► respuesta
```

### 2. Routing convencional y por atributos

```csharp
builder.Services.AddControllersWithViews();
// ...
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Pedidos}/{action=Index}/{id?}");
```

* `{controller=Pedidos}`: valor por defecto si la URL no lo trae.
* `{id?}`: segmento opcional.
* `/` equivale a `/Pedidos/Index`.

La alternativa es el routing por atributos, junto a la acción:

```csharp
[Route("pedidos")]
public sealed class PedidosController : Controller
{
    [HttpGet("{id:int}")]
    public IActionResult Detalle(int id) => ...;
}
```

| | Convencional | Por atributos |
| --- | --- | --- |
| Dónde se define | Un patrón global en `Program.cs` | Sobre cada controlador o acción |
| Ventaja | Consistencia sin repetir | URLs explícitas y restricciones por acción |
| Uso típico | Sitios MVC con vistas | APIs con `[ApiController]` |

### 3. Las convenciones que conectan todo

| Convención | Ejemplo |
| --- | --- |
| Sufijo `Controller` | `PedidosController` → `controller = "Pedidos"` |
| Carpeta de vistas | `Views/Pedidos/` |
| Nombre de la vista | La acción `Detalle` busca `Detalle.cshtml` |
| Vistas compartidas | `Views/Shared/` (layouts, parciales) |

`Controller` no es una palabra reservada de C#. Es una convención que ASP.NET Core usa para **descubrir** controladores: el compilador no sabe nada de ella. Por eso renombrar el controlador sin mover la carpeta de vistas rompe la búsqueda.

### 4. Responsabilidades

```text
Controlador   traduce HTTP ↔ aplicación: lee parámetros, llama a un caso de uso, elige vista o redirección
Caso de uso   ejecuta la operación del negocio
Dominio       protege reglas e invariantes
ViewModel     datos con la forma exacta que necesita la vista
Vista         presenta; no consulta la base ni decide reglas
```

Un controlador delgado es fácil de probar y de cambiar. El descuento del problema pertenece al dominio, no a la acción.

### 5. Model binding

MVC construye los parámetros de la acción desde la ruta, la query string, el formulario o el cuerpo:

```csharp
public IActionResult Buscar(string? texto, int pagina = 1)   // ?texto=teclado&pagina=2
```

En formularios HTML, el binding lee los campos por nombre. Después valida los atributos y deja el resultado en `ModelState`. **En MVC con vistas, nadie revisa `ModelState` por ti**: si es inválido, devuelve la vista con el modelo para mostrar los errores.

### 6. Razor codifica por defecto

```cshtml
<p>@Model.Cliente</p>
```

Si `Cliente` vale `<script>alert(1)</script>`, Razor escribe `&lt;script&gt;alert(1)&lt;/script&gt;`: el navegador lo muestra como texto. **`@Html.Raw(...)` desactiva esa protección.** Úsalo solo con HTML que tú generaste y sabes que es seguro, nunca con datos del usuario.

### 7. Tag Helpers

```cshtml
<a asp-action="Detalle" asp-route-id="@pedido.Id">Ver</a>
```

Genera `<a href="/Pedidos/Detalle/42">` a partir de la tabla de rutas. Si cambias el patrón de rutas, los enlaces se actualizan solos. Para activarlos, `Views/_ViewImports.cshtml` debe incluir:

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

En formularios, `<form asp-action="Crear" method="post">` agrega automáticamente el token antiforgery.

### 8. Formularios y verbos

Las acciones que cambian estado usan POST con validación antiforgery. **Nunca conviertas una eliminación en un enlace GET**: crawlers, el *prefetch* del navegador o un enlace en un correo podrían dispararla. El flujo completo de formularios con persistencia está en [CRUD seguro con MVC y EF Core](../../04-data-access-and-transactions/01-entity-framework-core/03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md).

-----

## Ejemplo completo

Proyecto `dotnet new web` con estos archivos.

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();
builder.Services.AddSingleton<IPedidoQuery, PedidoQueryEnMemoria>();

var app = builder.Build();
app.MapControllerRoute("default", "{controller=Pedidos}/{action=Index}/{id?}");
app.Run();

public sealed record PedidoFila(int Id, string Cliente, decimal Total);

public interface IPedidoQuery
{
    IReadOnlyList<PedidoFila> Listar();
    PedidoFila? Buscar(int id);
}

public sealed class PedidoQueryEnMemoria : IPedidoQuery
{
    private readonly PedidoFila[] _pedidos =
    [
        new(42, "Ana", 125.50m),
        new(43, "<script>alert(1)</script>", 80m)
    ];

    public IReadOnlyList<PedidoFila> Listar() => _pedidos;
    public PedidoFila? Buscar(int id) => _pedidos.FirstOrDefault(p => p.Id == id);
}
```

`Controllers/PedidosController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

public sealed class PedidosController(IPedidoQuery query) : Controller
{
    public IActionResult Index() => View(query.Listar());

    public IActionResult Detalle(int id) =>
        query.Buscar(id) is { } pedido ? View(pedido) : NotFound();
}
```

`Views/_ViewImports.cshtml`:

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

`Views/Pedidos/Index.cshtml`:

```cshtml
@model IReadOnlyList<PedidoFila>
<h1>Pedidos</h1>
<ul>
@foreach (var pedido in Model)
{
    <li><a asp-action="Detalle" asp-route-id="@pedido.Id">#@pedido.Id</a> @pedido.Cliente</li>
}
</ul>
```

`Views/Pedidos/Detalle.cshtml`:

```cshtml
@model PedidoFila
<h1>Pedido #@Model.Id</h1>
<p>Cliente: @Model.Cliente</p>
<p>Total: @Model.Total.ToString("F2")</p>
```

Peticiones:

```text
GET /                    → 200 HTML
GET /Pedidos/Detalle/42  → 200 HTML
GET /Pedidos/Detalle/99  → 404
```

HTML generado por `GET /` (fragmento):

```html
<li><a href="/Pedidos/Detalle/42">#42</a> Ana</li>
<li><a href="/Pedidos/Detalle/43">#43</a> &lt;script&gt;alert(1)&lt;/script&gt;</li>
```

Texto visible de `GET /Pedidos/Detalle/42` (con cultura `en-US`; en `es-ES` verás `125,50`):

```text
Pedido #42
Cliente: Ana
Total: 125.50
```

Qué observar:

* `/` llegó a `PedidosController.Index` por los valores por defecto del patrón.
* El Tag Helper generó los `href` desde la tabla de rutas: no hay URLs escritas a mano.
* El nombre malicioso se mostró como texto: Razor lo codificó.
* La vista recibe `PedidoFila`, no una entidad de base de datos.

-----

## Errores comunes

**1. Creer que `Controller` es una palabra reservada.**
Qué pasa: se atribuye al lenguaje una regla del framework y no se entiende por qué un renombrado rompe todo.
Por qué: el sufijo es una convención de descubrimiento de ASP.NET Core.
Arreglo: renombra controlador y carpeta de vistas juntos, o especifica la vista con `View("Ruta/Vista")`.

**2. Usar `@Html.Raw` con datos del usuario.**
Qué pasa: XSS: el navegador ejecuta el script que alguien guardó como nombre.
Por qué: `Html.Raw` desactiva la codificación HTML de Razor.
Arreglo: usa `@valor`; reserva `Html.Raw` para HTML generado por ti y ya saneado.

**3. Pasar entidades directamente a las vistas.**
Qué pasa: la vista queda acoplada al esquema, y modificar la entidad "para mostrarla" puede terminar guardado en la base.
Por qué: la entidad está rastreada por el `DbContext` y tiene la forma del modelo de datos, no de la pantalla.
Arreglo: proyecta a un ViewModel de solo lectura.

**4. No revisar `ModelState` en un POST.**
Qué pasa: se guardan datos que la validación ya había marcado como inválidos.
Por qué: en MVC con vistas, la acción se ejecuta aunque la validación falle.
Arreglo: `if (!ModelState.IsValid) return View(modelo);` antes de persistir.

**5. Modificar datos con GET.**
Qué pasa: un crawler o el *prefetch* del navegador borra registros.
Por qué: GET debe ser seguro; los agentes lo ejecutan sin pedir confirmación.
Arreglo: formularios con POST y antiforgery (`<form method="post">` con Tag Helpers).

-----

## Según la versión de .NET

* **ASP.NET MVC 5:** pertenecía a .NET Framework, sobre `System.Web`, con otro pipeline.
* **ASP.NET Core 1.0:** MVC se reescribe sobre el nuevo pipeline y se unifica con Web API.
* **ASP.NET Core 2.2 y 3.0:** aparece y se vuelve estándar el *endpoint routing*.
* **.NET 6:** el hosting mínimo permite registrar y mapear MVC desde `Program.cs`, sin `Startup.cs`.
* **.NET 10:** el modelo MVC no cambia; las plantillas usan `MapStaticAssets` y `WithStaticAssets()` para los archivos estáticos.

-----

## Cuándo sí y cuándo no

**Usa MVC con vistas cuando:** el servidor entrega HTML y varias acciones relacionadas comparten controlador (listado, detalle, edición de un mismo recurso).

**No lo uses cuando:** solo expones JSON (usa una API) o cada pantalla es un formulario independiente (Razor Pages lo organiza mejor).

**Usa rutas convencionales cuando:** el sitio sigue un patrón uniforme. **Usa atributos cuando:** necesitas URLs específicas por acción.

-----

## Resumen en 5 líneas

1. El routing traduce la URL a controlador, acción y parámetros.
2. Las convenciones (sufijo `Controller`, carpeta `Views/<Controlador>/`) conectan las piezas; no son reglas del compilador.
3. El controlador coordina; el negocio vive en el dominio y la vista recibe un ViewModel.
4. Razor codifica HTML por defecto; `Html.Raw` con datos del usuario es XSS.
5. Los cambios de estado usan POST con antiforgery y revisan `ModelState`.

-----

## Para profundizar

<details>
<summary>Layouts, secciones y vistas parciales</summary>

`Views/_ViewStart.cshtml` define el layout por defecto (`Layout = "_Layout";`). El layout tiene la estructura común (`<head>`, menú) y llama a `@RenderBody()` donde va cada vista. Las secciones (`@RenderSection("Scripts", required: false)`) permiten que una vista inyecte contenido en un lugar del layout. Las vistas parciales (`<partial name="_Resumen" model="..." />`) reutilizan fragmentos.

</details>

<details>
<summary>Áreas</summary>

En sitios grandes, las áreas agrupan controladores y vistas por módulo (`Areas/Admin/Controllers`, `Areas/Admin/Views`). El patrón de rutas incluye `{area:exists}`. Son una forma de organizar presentación; no reemplazan una separación real de módulos en el dominio.

</details>

-----

## En entrevista

### Respuesta corta (junior)

MVC separa modelo, vista y controlador. El routing elige el controlador y la acción según la URL; la acción obtiene los datos y devuelve una vista Razor que genera el HTML. Razor codifica el contenido para evitar XSS.

### Respuesta ampliada (semi-senior)

MVC es un patrón de presentación. Mantengo controladores delgados que delegan en casos de uso y entregan ViewModels, nunca entidades rastreadas. Conozco las convenciones de descubrimiento de controladores y vistas, y elijo entre rutas convencionales y por atributos según el sitio. En formularios reviso `ModelState`, uso POST con antiforgery y aplico Post/Redirect/Get. Evito `Html.Raw` con datos externos porque desactiva la codificación de Razor.

### Preguntas frecuentes de seguimiento

**1. ¿Dónde va la lógica de negocio?**
En el dominio o en casos de uso; ni en la vista ni concentrada en el controlador.

**2. ¿Por qué no borrar con GET?**
Porque GET debe ser seguro y los agentes lo ejecutan automáticamente.

**3. ¿Cómo encuentra MVC una vista?**
Busca `Views/<Controlador>/<Acción>.cshtml` y después `Views/Shared/<Acción>.cshtml`.

**4. ¿Razor protege contra XSS?**
Codifica la salida por defecto. `Html.Raw` desactiva esa protección.

-----

## Práctica

**Ejercicio 1.** Cambia la ruta por defecto para que el sitio inicie en `Catalogo/Index`, y agrega una ruta `/ofertas` que lleve a `CatalogoController.Ofertas` sin cambiar el patrón general.

<details>
<summary>Solución</summary>

```csharp
app.MapControllerRoute(
    name: "ofertas",
    pattern: "ofertas",
    defaults: new { controller = "Catalogo", action = "Ofertas" });

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Catalogo}/{action=Index}/{id?}");
```

La ruta específica va **antes** de la general. En las rutas convencionales el orden de registro define la prioridad, así que conviene ir de lo particular a lo general. Con el orden invertido, `/ofertas` coincidiría primero con el patrón general como `controller = "ofertas"`.

</details>

**Ejercicio 2.** Refactoriza el `HomeController` del problema: sin descuento sobre entidades, sin `Html.Raw` y con un ViewModel.

<details>
<summary>Solución</summary>

```csharp
public sealed record PedidoResumen(int Id, string Cliente, decimal TotalConDescuento);

public sealed class PedidosController(IPedidoConsultas consultas) : Controller
{
    public async Task<IActionResult> Index(CancellationToken cancellationToken) =>
        View(await consultas.ListarConDescuentoAsync(cancellationToken));
}
```

`IPedidoConsultas` proyecta a `PedidoResumen` con `AsNoTracking` y calcula el descuento con una regla del dominio. La vista muestra `@pedido.Cliente`, que Razor codifica. El controlador quedó en una línea: traduce HTTP, nada más.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md) si construirás servicios HTTP, o con [Entity Framework Core](../../04-data-access-and-transactions/01-entity-framework-core/README.md) para persistir datos. También puedes volver al [índice del módulo](README.md).
