# HATEOAS

## En una frase

**HATEOAS** (*Hypermedia As The Engine Of Application State*) significa que cada respuesta incluye **enlaces** a las acciones que el cliente **puede hacer ahora**, según el estado del recurso: un pedido pendiente trae los enlaces "pagar" y "cancelar"; uno enviado, no.

-----

## Antes de empezar

Conviene que ya sepas:

* Recursos, rutas y sub-recursos de acción: [Principios REST y códigos de estado](01-Principios%20REST%20y%20codigos%20de%20estado.md).
* Versionamiento por URL: [Versionamiento de APIs](03-Versionamiento%20de%20APIs.md).
* Estados de un agregado: [Modelo anémico y modelo de dominio](../../02-architecture-and-design-patterns/01-domain-driven-design/01-Modelo%20anemico%20y%20modelo%20de%20dominio.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Hipermedia:** contenido que incluye enlaces a otros recursos o acciones, como el HTML con sus `<a>` y `<form>`.
* **Enlace (*link*):** objeto con la URL (`href`), la relación (`rel`) y, a menudo, el método HTTP.
* **`rel` (relación):** nombre que indica qué significa el enlace: `self`, `next`, `cancelar`.
* **HAL / JSON:API / Siren:** formatos estándar para representar enlaces en JSON.
* **Nivel 3 de Richardson:** el nivel de madurez de una API REST que usa hipermedia.

-----

## El problema

El frontend muestra un pedido con sus botones. ¿Cuándo aparece "Cancelar"?

```javascript
// En el frontend
if (pedido.estado === "Pendiente" || (pedido.estado === "Pagado" && !pedido.enviado)) {
  mostrarBoton("Cancelar", () => fetch(`/api/pedidos/${pedido.id}/cancelar`, { method: "POST" }));
}
```

Dos problemas:

1. **La regla del negocio está duplicada.** El backend ya sabe cuándo se puede cancelar (lo valida el dominio). Si la regla cambia ("ahora se puede cancelar hasta 24 horas después de pagar"), hay que cambiarla en el backend, en la web, en la app de Android y en la de iOS, y desplegar todo a la vez.
2. **Las URLs están escritas a mano en cada cliente.** Si la ruta cambia, todos los clientes se rompen.

-----

## Cómo funciona

### 1. La respuesta dice qué se puede hacer

```json
{
  "id": 1,
  "cliente": "Ana",
  "total": 150.0,
  "estado": "Pendiente",
  "links": [
    { "href": "/api/pedidos/1",          "rel": "self",     "method": "GET"  },
    { "href": "/api/pedidos/1/pagar",    "rel": "pagar",    "method": "POST" },
    { "href": "/api/pedidos/1/cancelar", "rel": "cancelar", "method": "POST" }
  ]
}
```

El frontend ya no decide; solo pregunta si existe el enlace:

```javascript
const cancelar = pedido.links.find(l => l.rel === "cancelar");
if (cancelar) mostrarBoton("Cancelar", () => fetch(cancelar.href, { method: cancelar.method }));
```

Si la regla de cancelación cambia, **solo cambia el backend**.

### 2. Los enlaces dependen del estado

Aquí está el verdadero valor. Si todos los pedidos devuelven siempre `self`, `update` y `delete`, los enlaces no dicen nada que el cliente no supiera.

```text
                 pagar                    enviar
  ┌───────────┐ ───────► ┌───────────┐ ─────────► ┌───────────┐
  │ Pendiente │          │  Pagado   │            │  Enviado  │
  └───────────┘          └───────────┘            └───────────┘
        │ cancelar             │ cancelar
        ▼                      ▼
  ┌───────────┐          ┌───────────┐
  │ Cancelado │          │ Cancelado │
  └───────────┘          └───────────┘

  Enlaces por estado:
  Pendiente → self, pagar, cancelar
  Pagado    → self, enviar, cancelar
  Enviado   → self
  Cancelado → self
```

La API expone la **máquina de estados** del negocio: el cliente navega por ella siguiendo enlaces, como una persona navega por una web haciendo clic.

### 3. Generar las URLs, no escribirlas

```csharp
Url.Action(nameof(Obtener), new { id = pedido.Id })    // "/api/pedidos/1"
```

* **`Url.Action`** (en controladores) arma la URL a partir de la acción y los valores de ruta. Si cambias la ruta en el atributo `[Route]`, los enlaces se actualizan solos.
* **`LinkGenerator`** (inyectable) hace lo mismo fuera de los controladores: en servicios, filtros o minimal APIs.
* Con versionamiento por URL, incluye la versión: `new { id, version = RouteData.Values["version"] }`.

Escribir `$"/api/orders/{id}"` a mano es justamente lo que HATEOAS intenta evitar: si la ruta real es `/api/v1/orders/{id}`, el enlace queda roto.

### 4. Formatos de enlaces

| Formato | Forma | Comentario |
| --- | --- | --- |
| Propio (`links: [...]`) | `{ "href", "rel", "method" }` | Simple; el que usa esta lección |
| **HAL** | `"_links": { "self": { "href": "..." } }` | Muy extendido; no incluye el método HTTP |
| **JSON:API** | `"links": { "self": "..." }` + estructura completa del documento | Especificación grande y estricta |
| **Siren** | `"actions": [ { "name", "method", "href", "fields" } ]` | Incluye los campos de cada acción, como un formulario |

Lo importante no es el formato sino **ser consistente** en toda la API y documentar los `rel`.

### 5. Colecciones: enlaces de paginación

```json
{
  "items": [ ... ],
  "page": 2,
  "pageSize": 20,
  "totalCount": 135,
  "links": [
    { "href": "/api/pedidos?page=2&pageSize=20", "rel": "self", "method": "GET" },
    { "href": "/api/pedidos?page=1&pageSize=20", "rel": "prev", "method": "GET" },
    { "href": "/api/pedidos?page=3&pageSize=20", "rel": "next", "method": "GET" }
  ]
}
```

Cuando no hay página siguiente, no hay enlace `next`: el cliente sabe que terminó sin calcularlo.

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web`:

```csharp
using System.Collections.Concurrent;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddSingleton<PedidoStore>();

var app = builder.Build();
app.MapControllers();
app.Run();

public enum EstadoPedido { Pendiente, Pagado, Enviado, Cancelado }

public class Pedido(int id, string cliente, decimal total)
{
    public int Id { get; } = id;
    public string Cliente { get; } = cliente;
    public decimal Total { get; } = total;
    public EstadoPedido Estado { get; private set; } = EstadoPedido.Pendiente;

    public bool PuedePagar => Estado == EstadoPedido.Pendiente;
    public bool PuedeEnviar => Estado == EstadoPedido.Pagado;
    public bool PuedeCancelar => Estado is EstadoPedido.Pendiente or EstadoPedido.Pagado;

    public void Pagar() => Estado = PuedePagar ? EstadoPedido.Pagado : throw NoPermitido("pagar");
    public void Enviar() => Estado = PuedeEnviar ? EstadoPedido.Enviado : throw NoPermitido("enviar");
    public void Cancelar() => Estado = PuedeCancelar ? EstadoPedido.Cancelado : throw NoPermitido("cancelar");

    private InvalidOperationException NoPermitido(string accion) =>
        new($"No se puede {accion} un pedido en estado {Estado}.");
}

public class PedidoStore
{
    private readonly ConcurrentDictionary<int, Pedido> _pedidos = new()
    {
        [1] = new Pedido(1, "Ana", 150m),
        [2] = new Pedido(2, "Luis", 80m),
    };

    public Pedido? Buscar(int id) => _pedidos.GetValueOrDefault(id);
}

public record Enlace(string Href, string Rel, string Method);

public record PedidoDto(int Id, string Cliente, decimal Total, string Estado, IReadOnlyList<Enlace> Links);

[ApiController]
[Route("api/pedidos")]
public class PedidosController(PedidoStore store) : ControllerBase
{
    [HttpGet("{id:int}")]
    public ActionResult<PedidoDto> Obtener(int id) =>
        store.Buscar(id) is { } pedido ? ConEnlaces(pedido) : NotFound();

    [HttpPost("{id:int}/pagar")]
    public ActionResult<PedidoDto> Pagar(int id) => Ejecutar(id, p => p.Pagar());

    [HttpPost("{id:int}/enviar")]
    public ActionResult<PedidoDto> Enviar(int id) => Ejecutar(id, p => p.Enviar());

    [HttpPost("{id:int}/cancelar")]
    public ActionResult<PedidoDto> Cancelar(int id) => Ejecutar(id, p => p.Cancelar());

    private ActionResult<PedidoDto> Ejecutar(int id, Action<Pedido> accion)
    {
        var pedido = store.Buscar(id);
        if (pedido is null) return NotFound();

        try { accion(pedido); }
        catch (InvalidOperationException ex)
        {
            return Problem(title: "Operación no permitida", detail: ex.Message, statusCode: StatusCodes.Status409Conflict);
        }

        return ConEnlaces(pedido);
    }

    private PedidoDto ConEnlaces(Pedido p)
    {
        var enlaces = new List<Enlace> { Enlazar(nameof(Obtener), p.Id, "self", "GET") };
        if (p.PuedePagar) enlaces.Add(Enlazar(nameof(Pagar), p.Id, "pagar", "POST"));
        if (p.PuedeEnviar) enlaces.Add(Enlazar(nameof(Enviar), p.Id, "enviar", "POST"));
        if (p.PuedeCancelar) enlaces.Add(Enlazar(nameof(Cancelar), p.Id, "cancelar", "POST"));

        return new PedidoDto(p.Id, p.Cliente, p.Total, p.Estado.ToString(), enlaces);
    }

    private Enlace Enlazar(string accion, int id, string rel, string metodo) =>
        new(Url.Action(accion, new { id })!, rel, metodo);
}
```

Una secuencia de peticiones:

```text
GET  /api/pedidos/1           → 200 { "estado": "Pendiente", "links": [ self, pagar, cancelar ] }
POST /api/pedidos/1/pagar     → 200 { "estado": "Pagado",    "links": [ self, enviar, cancelar ] }
POST /api/pedidos/1/enviar    → 200 { "estado": "Enviado",   "links": [ self ] }
POST /api/pedidos/1/cancelar  → 409 { "title": "Operación no permitida",
                                      "detail": "No se puede cancelar un pedido en estado Enviado." }
```

Detalle de uno de los enlaces generados:

```json
{ "href": "/api/pedidos/1/enviar", "rel": "enviar", "method": "POST" }
```

Fíjate en dos cosas:

* Las reglas (`PuedePagar`, `PuedeCancelar`) viven en **un solo lugar**, el modelo, y la API las usa tanto para validar como para decidir qué enlaces mostrar.
* Ninguna URL está escrita a mano: si cambias `[Route("api/pedidos")]`, los enlaces cambian solos.

-----

## Errores comunes

**1. Enlaces fijos que no dependen del estado.**
Qué pasa: `self`, `update` y `delete` en todos los recursos, siempre; el cliente no aprende nada nuevo.
Por qué: se agregan enlaces "porque HATEOAS lo pide".
Arreglo: incluye solo las acciones que **ahora** están permitidas.

**2. URLs escritas a mano.**
Qué pasa: enlaces rotos cuando cambia la ruta o la versión (`/api/orders/1` cuando la ruta real es `/api/v1/orders/1`).
Por qué: interpolación de strings en lugar de generación.
Arreglo: `Url.Action` o `LinkGenerator`, con todos los valores de ruta (incluida la versión).

**3. Varios `rel` con el mismo `href` y sin método.**
Qué pasa: `update` y `delete` apuntan a `/orders/1`, y el cliente no sabe qué verbo usar.
Por qué: el enlace no incluye el método HTTP.
Arreglo: incluye `method`, o usa un formato que lo defina (Siren) o documenta cada `rel`.

**4. Duplicar las reglas para decidir los enlaces.**
Qué pasa: el controlador decide "si estado == Pendiente, agrego pagar" y el dominio valida otra cosa; los enlaces mienten.
Por qué: la regla se copió en vez de reutilizarse.
Arreglo: el modelo expone las capacidades (`PuedePagar`) y tanto la validación como los enlaces las usan.

**5. Creer que el cliente ya no necesita documentación.**
Qué pasa: el cliente recibe un `rel: "aprobar"` y no sabe qué cuerpo enviar ni qué significa.
Por qué: HATEOAS dice **qué** se puede hacer, no **cómo**.
Arreglo: documenta los `rel` y los cuerpos (OpenAPI), igual que siempre.

-----

## Según la versión de .NET

ASP.NET Core no trae soporte específico para HATEOAS; se construye con herramientas generales:

* **ASP.NET Core 1.0:** `IUrlHelper` (`Url.Action`, `Url.Link`).
* **ASP.NET Core 2.2:** `LinkGenerator`, usable fuera de los controladores con *endpoint routing*.
* **C# 9:** records, ideales para los DTOs de enlaces.
* Librerías de terceros para formatos como HAL existen, pero en la mayoría de los proyectos basta con un record `Enlace`.

-----

## Cuándo sí y cuándo no

**Vale la pena cuando:**

* El recurso tiene un **flujo de estados** con acciones que cambian según el estado (pedidos, solicitudes, aprobaciones, reservas).
* Tienes varios clientes (web, móvil) que hoy duplican reglas del negocio para mostrar u ocultar acciones.
* Paginación: los enlaces `next`/`prev` son útiles y baratos.

**No vale la pena cuando:**

* Es un CRUD sin estados: los enlaces serían siempre los mismos.
* El único cliente es tu propio frontend, que se despliega junto con la API, y las reglas son triviales.
* El tamaño de la respuesta es crítico (colecciones enormes con enlaces por elemento).

Honestamente: muy pocas APIs del mundo real son HATEOAS "puro" (nivel 3 de Richardson). Lo habitual es usarlo de forma **pragmática** en los recursos con flujo de estados y en la paginación.

-----

## Resumen en 5 líneas

1. HATEOAS: cada respuesta incluye enlaces a las acciones disponibles en el estado actual.
2. Su valor está en los enlaces que dependen del estado; si son fijos, no aportan.
3. Genera las URLs con `Url.Action` o `LinkGenerator`, nunca a mano.
4. Las reglas viven en el modelo y la API las usa para validar y para decidir los enlaces.
5. Úsalo de forma pragmática: flujos de estados y paginación, no en cada CRUD.

-----

## Para profundizar

<details>
<summary>Enlaces y permisos</summary>

Los enlaces pueden reflejar también **quién** pregunta: un usuario sin permiso de aprobación no recibe el enlace `aprobar`. Así el frontend oculta el botón sin conocer la política de permisos. La validación del permiso en el endpoint sigue siendo obligatoria: el enlace es una guía, no una barrera de seguridad.

</details>

<details>
<summary>LinkGenerator fuera de los controladores</summary>

```csharp
public class EnlacesDePedido(LinkGenerator links, IHttpContextAccessor accessor)
{
    public string? Self(int id) =>
        links.GetPathByAction(accessor.HttpContext!, action: "Obtener", controller: "Pedidos", values: new { id });
}
```

Requiere `builder.Services.AddHttpContextAccessor()`. Para URLs absolutas, `GetUriByAction`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

HATEOAS es incluir enlaces en las respuestas de la API que indican qué acciones se pueden hacer con ese recurso. Por ejemplo, un pedido pendiente trae el enlace para pagarlo o cancelarlo, y uno enviado no. Así el frontend no tiene que saber las reglas: si el enlace está, muestra el botón.

### Respuesta ampliada (semi-senior)

HATEOAS es el nivel 3 del modelo de Richardson: la API expone su máquina de estados mediante enlaces con `rel`, `href` y método, que dependen del estado del recurso y, opcionalmente, de los permisos del usuario. Eso evita duplicar reglas del negocio en cada cliente y desacopla a los clientes de las URLs. Genero los enlaces con `Url.Action` o `LinkGenerator` a partir de capacidades que expone el dominio, para no duplicar reglas. Lo aplico de forma pragmática: en recursos con flujos de estados y en paginación, no en CRUDs simples, porque agrega tamaño y complejidad, y los clientes igual necesitan documentación de los `rel` y los cuerpos.

### Preguntas frecuentes de seguimiento

**1. ¿Qué ventaja real tiene HATEOAS?**
Que las reglas de qué acciones están disponibles viven solo en el backend, y los clientes no escriben URLs a mano.

**2. ¿Por qué no es tan común?**
Porque requiere clientes que realmente sigan los enlaces y agrega tamaño a las respuestas; muchos equipos prefieren documentación OpenAPI y clientes generados.

**3. ¿Un enlace oculto reemplaza la validación?**
No. El endpoint debe validar siempre; el enlace es solo una guía para el cliente.

-----

## Práctica

**Ejercicio 1.** Una solicitud de vacaciones tiene los estados `Borrador`, `Enviada`, `Aprobada`, `Rechazada`. Desde `Borrador` se puede `enviar` o `eliminar`; desde `Enviada`, `aprobar` o `rechazar`. Escribe qué enlaces (además de `self`) devuelve cada estado.

<details>
<summary>Solución</summary>

```text
Borrador  → self, enviar (POST), eliminar (DELETE)
Enviada   → self, aprobar (POST), rechazar (POST)
Aprobada  → self
Rechazada → self
```

Si además solo un jefe puede aprobar, un empleado consultando su propia solicitud `Enviada` recibe solo `self`.

</details>

**Ejercicio 2.** Escribe un método que arme los enlaces de paginación (`self`, `prev`, `next`) para `GET /api/pedidos?page={page}&pageSize={pageSize}`, sabiendo el `totalCount`.

<details>
<summary>Solución</summary>

```csharp
private List<Enlace> EnlacesDePagina(int page, int pageSize, int totalCount)
{
    var totalPaginas = (int)Math.Ceiling(totalCount / (double)pageSize);
    var enlaces = new List<Enlace> { Pagina(page, "self") };
    if (page > 1) enlaces.Add(Pagina(page - 1, "prev"));
    if (page < totalPaginas) enlaces.Add(Pagina(page + 1, "next"));
    return enlaces;

    Enlace Pagina(int numero, string rel) =>
        new(Url.Action(nameof(Listar), new { page = numero, pageSize })!, rel, "GET");
}
```

Los valores que no son parámetros de la ruta (`page`, `pageSize`) se agregan solos como query string.

</details>

-----

## Siguiente lección

Terminaste el módulo. Para practicar con los ejercicios de clase: [Ejercicios de diseño de APIs](Ejercicios.md). Vuelve al [índice del módulo](README.md).
