# MVC, routing y vistas Razor

## En una frase
MVC enruta una solicitud hacia una acción, obtiene un modelo de presentación y renderiza una vista Razor sin mezclar la interfaz con la lógica de negocio.

-----

## Antes de empezar
Conviene que ya sepas: [Modelos de aplicaciones ASP.NET Core](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md).

Palabras nuevas: **action:** método invocable por routing; **model binding:** construcción de parámetros desde HTTP; **ViewModel:** datos diseñados para una vista.

-----

## El problema
Un controlador que consulta EF Core, calcula descuentos y forma HTML funciona al principio, pero acopla transporte, negocio, persistencia y presentación. Renombrar `HomeController` sin actualizar ruta y carpeta de vistas también produce fallos de búsqueda.

-----

## Cómo funciona
### 1. Registro y rutas
```csharp
builder.Services.AddControllersWithViews();
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

### 2. Convenciones
`OrdersController.Index` busca normalmente `Views/Orders/Index.cshtml`. `Controller` no es palabra reservada de C#; es una convención de nombres del framework y sus herramientas.

### 3. Responsabilidades
El controlador traduce HTTP y coordina. Un caso de uso implementa la operación. La vista presenta un ViewModel. El dominio protege reglas.

### 4. Model binding y validación
MVC enlaza ruta, query, formularios y body según el tipo de acción. En formularios, valida `ModelState` antes de persistir y vuelve a mostrar errores.

### 5. Razor y Tag Helpers
Las vistas `.cshtml` combinan HTML con expresiones Razor. Tag Helpers como `asp-for` generan nombres y atributos basados en el modelo, pero no reemplazan validación del servidor.

### 6. Seguridad de formularios
Las acciones que cambian estado usan POST y protección antiforgery. Nunca conviertas una eliminación en GET porque enlaces, crawlers o prefetch podrían dispararla.

-----

## Ejemplo completo
`Program.cs`:
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();
builder.Services.AddSingleton<IOrderQuery, DemoOrderQuery>();

var app = builder.Build();
app.MapControllerRoute("default", "{controller=Orders}/{action=Index}/{id?}");
app.Run();

public sealed record OrderRow(Guid Id, decimal Total);
public interface IOrderQuery { IReadOnlyList<OrderRow> GetAll(); }
public sealed class DemoOrderQuery : IOrderQuery
{
    public IReadOnlyList<OrderRow> GetAll() => [new(Guid.Parse("11111111-1111-1111-1111-111111111111"), 125.50m)];
}
```

`Controllers/OrdersController.cs`:
```csharp
using Microsoft.AspNetCore.Mvc;

public sealed class OrdersController(IOrderQuery query) : Controller
{
    public IActionResult Index() => View(query.GetAll());
}
```

`Views/Orders/Index.cshtml`:
```cshtml
@model IReadOnlyList<OrderRow>
<h1>Pedidos</h1>
@foreach (var order in Model)
{
    <p>@order.Id — @order.Total.ToString("F2")</p>
}
```

Salida visible con una cultura que usa punto decimal:
```text
Pedidos
11111111-1111-1111-1111-111111111111 — 125.50
```

-----

## Errores comunes
**1. Creer que `Controller` es palabra reservada.** Qué pasa: se atribuye al lenguaje una convención. Por qué: C# no reserva ese sufijo. Arreglo: distingue compilador y descubrimiento MVC.

**2. Pasar entidades directamente a vistas.** Qué pasa: la UI queda acoplada al esquema persistente. Por qué: una entidad no expresa necesidades específicas de presentación. Arreglo: usa ViewModels cuando los contratos divergen.

**3. Modificar datos con GET.** Qué pasa: una navegación involuntaria cambia estado. Por qué: GET debe ser seguro. Arreglo: usa POST/PUT/DELETE y antiforgery donde corresponda.

-----

## Según la versión de .NET
- **ASP.NET MVC 5:** pertenecía a .NET Framework y usaba otro pipeline.
- **ASP.NET Core 1-2:** separaba configuración habitualmente en `Startup.cs`.
- **ASP.NET Core 3+:** usa endpoint routing.
- **.NET 6-10:** el hosting mínimo permite registrar y mapear MVC desde `Program.cs`.

-----

## Cuándo sí y cuándo no
**Usa MVC con vistas cuando:** el servidor debe entregar HTML y la separación controlador/vista ayuda al equipo. **No lo uses cuando:** solo expones JSON; una API evita renderizado y estructura innecesarios.

-----

## Resumen en 5 líneas
1. Routing selecciona un controlador y una acción.
2. El controlador coordina HTTP, no concentra el negocio.
3. Las vistas Razor presentan modelos fuertemente tipados.
4. Model binding no sustituye validación ni autorización.
5. Los cambios de estado usan verbos adecuados y protección antiforgery.

-----

## Para profundizar
<details><summary>Ruta convencional frente a atributos</summary>La ruta convencional aplica un patrón global; los atributos colocan la plantilla junto a la acción. Puedes combinarlos con criterio, pero evita que una misma área sea impredecible.</details>

-----

## En entrevista
### Respuesta corta (junior)
MVC separa modelo, vista y controlador. Routing elige la acción, el controlador obtiene datos y la vista Razor genera HTML.

### Respuesta ampliada (semi-senior)
MVC es una arquitectura de presentación. El controlador debe delegar casos de uso, usar ViewModels y tratar correctamente binding, validación, autorización y antiforgery. El dominio no debe depender de MVC.

### Preguntas frecuentes de seguimiento
**1. ¿Dónde va la lógica de negocio?** En dominio o casos de uso, no en la vista ni concentrada en el controlador.

**2. ¿Por qué no borrar con GET?** Porque GET debe ser seguro y puede ejecutarse automáticamente.

-----

## Práctica
**Ejercicio 1.** Cambia la ruta predeterminada para iniciar en `Catalog/Index`.
<details><summary>Solución</summary>

```csharp
app.MapControllerRoute("default", "{controller=Catalog}/{action=Index}/{id?}");
```
</details>

-----

## Siguiente lección
[Diseño de APIs REST](../01-diseno-de-apis-rest/README.md) · [Volver al índice](README.md)
