# Controladores y rutas por atributo

## En una frase

En una Web API con controladores, la ruta de cada endpoint sale de combinar el `[Route]` del controlador con la plantilla del atributo de verbo (`[HttpGet("{id:int}")]`); las rutas nombran **recursos estables**, no métodos de C#, y el tipo de retorno `ActionResult<T>` documenta qué devuelve cada acción.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es `[ApiController]` y cómo se elige entre controladores y Minimal APIs: [Modelos de aplicaciones ASP.NET Core](../00-fundamentos-de-aspnet-core/02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md).
* Recursos, verbos y códigos de estado: [Principios REST y códigos de estado](../01-diseno-de-apis-rest/01-Principios%20REST%20y%20codigos%20de%20estado.md).
* Routing convencional frente a routing por atributos: [MVC, routing y vistas Razor](../00-fundamentos-de-aspnet-core/03-MVC%20routing%20y%20vistas%20Razor.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Plantilla de ruta:** texto como `api/tareas/{id:int}` que define qué URLs coinciden con un endpoint.
* **Token de reemplazo:** marcador `[controller]` o `[action]` que ASP.NET Core sustituye por el nombre del controlador o del método.
* **Restricción de ruta:** condición sobre un parámetro, como `{id:int}`, que decide si la URL coincide.
* **`ActionResult<T>`:** tipo de retorno que permite devolver un `T` (200) o cualquier otro resultado HTTP, y que documenta el tipo del cuerpo.

-----

## El problema

Un controlador armado con lo que aparece en muchos tutoriales:

```csharp
[ApiController]
[Route("api/[controller]")]
public class TareasController : ControllerBase
{
    [HttpGet("[action]")]
    [HttpGet("todos")]
    public IActionResult Listar() => Ok(_servicio.Listar());

    [HttpGet("Get/{id}")]
    public IActionResult Obtener(string id) => Ok(_servicio.Obtener(int.Parse(id)));

    [HttpGet("{nombre}")]
    public IActionResult BuscarPorNombre(string nombre) => Ok(_servicio.Buscar(nombre));
}
```

Lo que pasa:

1. Alguien renombra `Listar` a `ObtenerTodas` en un refactor. La ruta `api/Tareas/Listar` **desaparece**, porque `[action]` la ataba al nombre del método. La app móvil empieza a recibir `404`.
2. `GET api/Tareas/Get/abc` lanza `FormatException` en `int.Parse`: el cliente recibe `500` por un error suyo.
3. `GET api/Tareas/todos` coincide con `[HttpGet("todos")]` **y** con `[HttpGet("{nombre}")]`. ASP.NET Core prefiere la ruta literal, así que funciona... hasta que alguien crea una tarea llamada "todos" y nunca puede buscarla.
4. Todas las acciones devuelven `IActionResult`: la documentación OpenAPI no sabe qué JSON devuelve ninguna.

Las consecuencias:

* rutas que cambian por refactors internos;
* `500` por datos inválidos del cliente que debían ser `404` o `400`;
* contratos de API que no se pueden documentar ni generar automáticamente.

-----

## Cómo funciona

### 1. Qué hace `[ApiController]`

| Efecto | Sin el atributo | Con el atributo |
| --- | --- | --- |
| Routing | Convencional o por atributos | **Obligatorio** por atributos |
| Modelo inválido | Tienes que comprobar `ModelState` | `400` automático con `ValidationProblemDetails` |
| Origen de parámetros | Hay que indicar `[FromBody]` en tipos complejos | Inferido: complejos del cuerpo, simples de la ruta o query |
| Errores `4xx` sin cuerpo | Sin cuerpo | `ProblemDetails` |

Ponlo en todos los controladores de API, o una vez para el ensamblado con `[assembly: ApiController]`.

### 2. Cómo se arma la ruta final

```text
[Route("api/tareas")]         en el controlador
     +
[HttpGet("{id:int}")]         en la acción
     =
GET api/tareas/{id:int}
```

| Plantilla en la acción | Ruta final |
| --- | --- |
| `[HttpGet]` | `api/tareas` |
| `[HttpGet("{id:int}")]` | `api/tareas/5` |
| `[HttpGet("{id:int}/historial")]` | `api/tareas/5/historial` |
| `[HttpGet("/salud")]` | `salud` (la `/` inicial **ignora** el prefijo del controlador) |

### 3. Tokens: `[controller]` y `[action]`

```csharp
[Route("api/[controller]")]   // TareasController → api/Tareas
```

* `[controller]` es el nombre de la clase sin el sufijo `Controller`. Es cómodo, pero renombrar la clase cambia la URL pública.
* `[action]` es el nombre del método. Ata la URL a un detalle de implementación y suele producir rutas con verbos (`api/Tareas/ObtenerTodas`). **Evítalo en APIs REST.**

Una ruta literal (`[Route("api/tareas")]`) es más larga de escribir y nunca cambia por accidente. Para una API pública, ese es el criterio que importa.

### 4. Restricciones de ruta

```csharp
[HttpGet("{id:int}")]                  // solo enteros
[HttpGet("{id:guid}")]                 // solo Guid
[HttpGet("{codigo:alpha:length(3)}")]  // tres letras
```

Si el valor no cumple la restricción, la ruta **no coincide** y la respuesta es `404`, no `500`. El parámetro llega ya convertido (`int id`), sin `int.Parse`.

Las restricciones sirven para **elegir rutas**, no para validar reglas de negocio: `{edad:min(18)}` responde `404` a un menor de edad, cuando el código correcto sería `400`.

### 5. Varias rutas para una acción

```csharp
[HttpGet("pendientes")]
[HttpGet("abiertas")]   // alias temporal mientras los clientes migran
public ActionResult<List<TareaDto>> Pendientes() => ...
```

Es válido, pero cada ruta es un contrato que hay que mantener y documentar. Úsalo para migraciones con fecha de fin, no como costumbre.

### 6. Tipos de retorno

| Tipo | Ejemplo | OpenAPI conoce el cuerpo | Cuándo |
| --- | --- | --- | --- |
| `T` | `public TareaDto Obtener()` | Sí | Siempre 200 (raro en la práctica) |
| `IActionResult` | `return Ok(dto);` | **No** (necesitas `[ProducesResponseType]`) | Acciones sin cuerpo: `204`, `404` |
| `ActionResult<T>` | `return dto;` o `return NotFound();` | Sí | **Por defecto** cuando hay cuerpo |

```csharp
[HttpGet("{id:int}")]
public ActionResult<TareaDto> Obtener(int id)
{
    TareaDto? tarea = servicio.Obtener(id);
    if (tarea is null)
    {
        return NotFound();
    }
    return tarea;   // conversión implícita a ActionResult<TareaDto>: 200 con el cuerpo
}
```

La conversión implícita no funciona con interfaces: `ActionResult<IEnumerable<T>>` no acepta un `List<T>` directo. Devuelve el tipo concreto (`ActionResult<List<T>>`) o envuélvelo en `Ok(...)`.

### 7. Controladores delgados

El controlador traduce entre HTTP y la aplicación: lee la entrada, llama a un servicio y elige el código de estado. Las reglas de negocio viven en el servicio (o en el dominio), que no sabe nada de HTTP:

```csharp
[ApiController]
[Route("api/tareas")]
public sealed class TareasController(ITareaService servicio) : ControllerBase
{
    // ... acciones de 3 a 8 líneas
}
```

El constructor primario (C# 12) recibe la dependencia del contenedor de DI ([lección 2](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md)).

### 8. Mayúsculas en las URLs

El *matching* de rutas no distingue mayúsculas: `api/TAREAS/1` y `api/tareas/1` llegan al mismo endpoint. Lo que sí sale con mayúsculas son las URLs **generadas** (por ejemplo, el `Location` de `CreatedAtAction` con `[controller]`). Para generarlas en minúsculas:

```csharp
builder.Services.Configure<RouteOptions>(o => o.LowercaseUrls = true);
```

-----

## Ejemplo completo

Proyecto `dotnet new web`. `Program.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.Configure<RouteOptions>(o => o.LowercaseUrls = true);
builder.Services.AddSingleton<ITareaService, TareaServiceEnMemoria>();

var app = builder.Build();
app.MapControllers();
app.Run();

[ApiController]
[Route("api/tareas")]
public sealed class TareasController(ITareaService servicio) : ControllerBase
{
    [HttpGet]
    public ActionResult<List<TareaDto>> Listar() => servicio.Listar();

    [HttpGet("{id:int}")]
    public ActionResult<TareaDto> Obtener(int id)
    {
        TareaDto? tarea = servicio.Obtener(id);
        if (tarea is null)
        {
            return NotFound();
        }
        return tarea;
    }

    [HttpGet("pendientes")]
    public ActionResult<List<TareaDto>> Pendientes() =>
        servicio.Listar().Where(t => !t.Completada).ToList();

    [HttpPost]
    public ActionResult<TareaDto> Crear(CrearTareaRequest request)
    {
        TareaDto tarea = servicio.Crear(request.Titulo);
        return CreatedAtAction(nameof(Obtener), new { id = tarea.Id }, tarea);
    }

    [HttpPost("{id:int}/completar")]
    public IActionResult Completar(int id) =>
        servicio.Completar(id) ? NoContent() : NotFound();
}

public sealed record TareaDto(int Id, string Titulo, bool Completada);

public sealed record CrearTareaRequest([Required, MaxLength(100)] string Titulo);

public interface ITareaService
{
    List<TareaDto> Listar();
    TareaDto? Obtener(int id);
    TareaDto Crear(string titulo);
    bool Completar(int id);
}

// Singleton con estado compartido: el lock lo hace seguro entre peticiones concurrentes
public sealed class TareaServiceEnMemoria : ITareaService
{
    private readonly Lock _lock = new();
    private readonly List<TareaDto> _tareas = [new(1, "Pagar la luz", true), new(2, "Revisar PR", false)];
    private int _siguienteId = 3;

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
            var tarea = new TareaDto(_siguienteId++, titulo.Trim(), false);
            _tareas.Add(tarea);
            return tarea;
        }
    }

    public bool Completar(int id)
    {
        lock (_lock)
        {
            int indice = _tareas.FindIndex(t => t.Id == id);
            if (indice < 0)
            {
                return false;
            }
            _tareas[indice] = _tareas[indice] with { Completada = true };
            return true;
        }
    }
}
```

Peticiones (el puerto cambia según tu `launchSettings.json`):

```text
GET  /api/tareas                 → 200 [{"id":1,"titulo":"Pagar la luz","completada":true},{"id":2,"titulo":"Revisar PR","completada":false}]
GET  /api/tareas/2               → 200 {"id":2,"titulo":"Revisar PR","completada":false}
GET  /api/tareas/abc             → 404   (no cumple {id:int}: ninguna ruta coincide)
GET  /api/TAREAS/2               → 200   (el matching no distingue mayúsculas)
GET  /api/tareas/pendientes      → 200 [{"id":2,"titulo":"Revisar PR","completada":false}]
POST /api/tareas {"titulo":"Barrer"}
                                 → 201 {"id":3,"titulo":"Barrer","completada":false}
                                   Location: http://localhost:{puerto}/api/tareas/3
POST /api/tareas {"titulo":""}   → 400 ValidationProblemDetails (automático por [ApiController])
POST /api/tareas/99/completar    → 404
```

Qué observar:

* `abc` no llegó a la acción: la restricción `{id:int}` hizo que ninguna ruta coincidiera. No hubo `int.Parse` ni `500`.
* `pendientes` y `{id:int}` conviven sin ambigüedad porque `pendientes` no es un entero.
* El `Location` sale en minúsculas por `LowercaseUrls`.
* El `400` del título vacío lo generó `[ApiController]` antes de entrar a `Crear`.
* El servicio es **singleton** a propósito: guarda el estado en memoria para toda la aplicación, y por eso necesita el `lock` (`System.Threading.Lock`, C# 13).

-----

## Errores comunes

**1. Usar `[action]` en rutas de API.**
Qué pasa: renombrar un método cambia la URL pública y rompe clientes.
Por qué: el token se reemplaza por el nombre del método en cada compilación.
Arreglo: rutas literales por recurso; el verbo HTTP expresa la acción.

**2. Parámetros de ruta como `string` y conversión manual.**
Qué pasa: `FormatException` y `500` con valores inválidos.
Por qué: la conversión ocurre dentro de la acción, fuera del manejo de routing.
Arreglo: parámetro tipado (`int id`) con restricción (`{id:int}`).

**3. Dos rutas que coinciden con la misma URL.**
Qué pasa: `AmbiguousMatchException: The request matched multiple endpoints`.
Por qué: dos plantillas con la misma prioridad aceptan la URL (por ejemplo, `{id}` y `{nombre}`).
Arreglo: restricciones de tipo o segmentos distintos (`buscar?nombre=...`).

**4. `IActionResult` en acciones que devuelven un cuerpo.**
Qué pasa: OpenAPI no documenta el esquema de respuesta y los clientes generados reciben `object`.
Por qué: `IActionResult` no lleva información de tipo.
Arreglo: `ActionResult<T>`, o `[ProducesResponseType<T>(200)]` si debes usar `IActionResult`.

**5. Lógica de negocio en el controlador.**
Qué pasa: reglas duplicadas entre controladores y que no se pueden probar sin HTTP.
Por qué: el controlador mezcla traducción HTTP y decisiones del dominio.
Arreglo: el controlador llama a un servicio; el servicio no conoce `IActionResult`.

-----

## Según la versión de .NET

* **ASP.NET Core 1.0:** controladores unificados para MVC y Web API (`Controller` y `ControllerBase`).
* **ASP.NET Core 2.1:** `[ApiController]` y `ActionResult<T>`.
* **ASP.NET Core 2.2:** *endpoint routing*.
* **ASP.NET Core 3.0:** `MapControllers()`; el routing de controladores pasa a ser el mismo que el de cualquier endpoint.
* **ASP.NET Core 7:** `[ProducesResponseType<T>]` genérico.
* **C# 12:** constructores primarios en controladores. **C# 13:** tipo `System.Threading.Lock`.

-----

## Cuándo sí y cuándo no

**Usa controladores cuando:** el equipo ya los conoce, necesitas filtros de MVC, convenciones de aplicación o `ApiExplorer` con atributos, o la API tiene muchos endpoints con comportamiento común.

**Usa Minimal APIs cuando:** prefieres menos ceremonia y organizas por grupos de rutas (`MapGroup`) y archivos por recurso. Escalan bien; "Minimal API es solo para proyectos chicos" es un mito ([Endpoints y grupos de rutas](../02-minimal-apis/01-Endpoints%20y%20grupos%20de%20rutas.md)).

**No uses tokens `[action]` ni rutas con verbos cuando:** diseñas una API pública o consumida por otros equipos.

-----

## Resumen en 5 líneas

1. Ruta final = `[Route]` del controlador + plantilla del atributo de verbo.
2. Rutas literales por recurso; `[action]` ata la URL a un nombre de método.
3. Restricciones (`{id:int}`) eligen rutas y convierten parámetros: valores inválidos dan `404`, no `500`.
4. `ActionResult<T>` por defecto cuando hay cuerpo: documenta el tipo y permite otros códigos.
5. Controladores delgados: HTTP afuera, reglas en servicios.

-----

## Para profundizar

<details>
<summary>Orden de prioridad entre rutas</summary>

Cuando varias plantillas podrían coincidir, ASP.NET Core ordena por especificidad: los segmentos literales (`pendientes`) ganan a los parámetros con restricción (`{id:int}`), y estos a los parámetros sin restricción (`{nombre}`); los *catch-all* (`{**resto}`) van al final. Si después de ese orden quedan dos candidatos iguales, lanza `AmbiguousMatchException`. Puedes forzar el orden con la propiedad `Order` del atributo, pero si lo necesitas, probablemente las rutas están mal diseñadas.

</details>

<details>
<summary><code>[controller]</code> con transformadores</summary>

Un `IOutboundParameterTransformer` registrado como `RouteTokenTransformerConvention` puede convertir `[controller]` a *kebab-case* (`OrdenesDeCompraController` → `ordenes-de-compra`). Sigue teniendo el problema de que renombrar la clase cambia la URL; por eso muchas guías de APIs públicas prefieren rutas literales.

</details>

-----

## En entrevista

### Respuesta corta (junior)

En una Web API la ruta se arma con `[Route]` en el controlador y la plantilla del `[HttpGet]`, `[HttpPost]`, etc. en cada acción. Uso restricciones como `{id:int}` y devuelvo `ActionResult<T>` para poder responder el objeto o `NotFound()`.

### Respuesta ampliada (semi-senior)

Uso `[ApiController]` por la validación automática y los ProblemDetails, rutas literales por recurso (evito `[action]` porque ata la URL al nombre del método) y restricciones de tipo para que un valor inválido dé `404` y no `500`. Devuelvo `ActionResult<T>` para que OpenAPI conozca el esquema, `CreatedAtAction` para el `201` con `Location`, y mantengo los controladores delgados delegando en servicios inyectados. Sé que el matching no distingue mayúsculas y que `LowercaseUrls` solo afecta las URLs generadas.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `IActionResult` y `ActionResult<T>`?**
`ActionResult<T>` permite devolver el `T` directamente y documenta el tipo del cuerpo; `IActionResult` no lleva tipo.

**2. ¿Qué pasa si `{id:int}` recibe "abc"?**
La ruta no coincide y la respuesta es `404`.

**3. ¿Para qué sirve una `/` al inicio de la plantilla de una acción?**
Para ignorar el prefijo del controlador.

**4. ¿Por qué evitar `[action]`?**
Porque renombrar el método cambia la URL pública.

-----

## Práctica

**Ejercicio 1.** Corrige las rutas del controlador del problema para que sean REST, estables y sin ambigüedad.

<details>
<summary>Solución</summary>

```csharp
[ApiController]
[Route("api/tareas")]
public sealed class TareasController(ITareaService servicio) : ControllerBase
{
    [HttpGet]
    public ActionResult<List<TareaDto>> Listar([FromQuery] string? nombre) =>
        nombre is null ? servicio.Listar() : servicio.Buscar(nombre);

    [HttpGet("{id:int}")]
    public ActionResult<TareaDto> Obtener(int id)
    {
        TareaDto? tarea = servicio.Obtener(id);
        if (tarea is null)
        {
            return NotFound();
        }
        return tarea;
    }
}
```

* `[action]` y `todos` desaparecen: listar es `GET api/tareas`.
* La búsqueda por nombre pasa a la *query string* (`GET api/tareas?nombre=luz`): ya no compite con `{id}`.
* `int id` con `{id:int}` reemplaza el `string` + `int.Parse`.

</details>

**Ejercicio 2.** Agrega `GET api/tareas/{id}/historial` que devuelva una lista de strings, o `404` si la tarea no existe.

<details>
<summary>Solución</summary>

```csharp
[HttpGet("{id:int}/historial")]
public ActionResult<List<string>> Historial(int id)
{
    if (servicio.Obtener(id) is null)
    {
        return NotFound();
    }
    return new List<string> { "Creada", "Editada" };
}
```

La ruta final es `api/tareas/{id:int}/historial`: el segmento literal `historial` se suma al prefijo del controlador. `ActionResult<List<string>>` acepta la lista directamente porque `List<string>` es un tipo concreto.

</details>

-----

## Siguiente lección

[Inyección de dependencias y ciclos de vida](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md)
