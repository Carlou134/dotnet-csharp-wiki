# Principios REST y códigos de estado

## En una frase

Una API REST expone **recursos** (sustantivos, como `/pedidos`) que se manipulan con los **métodos HTTP** (GET, POST, PUT, PATCH, DELETE) y responde con **códigos de estado** que dicen qué pasó; el resultado es un **contrato** predecible que cualquier cliente sabe consumir sin leer tu código.

-----

## Antes de empezar

Conviene que ya sepas:

* Clases, records e interfaces: [C# Core](../../01-csharp-core-and-runtime/README.md).
* `async`/`await`: [Programación asíncrona](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/01-Programacion%20asincrona.md).
* Crear un proyecto: `dotnet new web -n PedidosApi` (ASP.NET Core vacío, .NET 10).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **API (*Application Programming Interface*):** contrato que permite a un programa usar las funciones de otro.
* **REST (*Representational State Transfer*):** estilo de arquitectura para APIs sobre HTTP basado en recursos.
* **Recurso:** cualquier "cosa" que la API expone y que tiene una dirección (URI): un pedido, un cliente, la lista de pedidos.
* **Endpoint:** combinación de método HTTP + ruta (`GET /api/pedidos/5`).
* **Código de estado:** número de 3 dígitos de la respuesta HTTP que indica el resultado (200, 404, 500...).
* **Idempotente:** repetir la misma petición una o diez veces deja el servidor en el mismo estado.
* **DTO (*Data Transfer Object*):** clase o record que define la forma de los datos que entran o salen de la API.

-----

## El problema

Una API "que funciona" pero sin criterio:

```http
POST /api/getPedido          { "id": 5 }
POST /api/crearPedido        { ... }
GET  /api/borrarPedido?id=5
POST /api/pedidos/actualizar { ... }
```

Y las respuestas:

```json
200 OK   { "ok": false, "error": "No existe" }
200 OK   "Error"
500      { "mensaje": "Object reference not set to an instance of an object." }
```

Problemas:

* **Cada endpoint es una sorpresa:** el frontend tiene que leer la documentación (o tu código) para saber qué verbo usar y qué forma tiene cada respuesta.
* **`GET /borrarPedido`:** un navegador, un *crawler* o un *prefetch* pueden **borrar datos** con solo visitar el enlace, porque GET se considera seguro.
* **`200 OK` con un error adentro:** las herramientas (cachés, monitoreo, reintentos, `fetch`, `HttpClient`) creen que todo salió bien.
* **El 500 filtra detalles internos** y no le dice al cliente qué hacer.

-----

## Cómo funciona

### 1. Recursos con sustantivos, acciones con métodos HTTP

```text
              Recurso             GET            POST           PUT / PATCH        DELETE
  ┌───────────────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐  ┌──────────┐
  │ /api/pedidos          │  │ listar     │  │ crear      │  │      —         │  │    —     │
  │ /api/pedidos/5        │  │ obtener 5  │  │     —      │  │ reemplazar /   │  │ borrar 5 │
  │                       │  │            │  │            │  │ modificar 5    │  │          │
  │ /api/pedidos/5/lineas │  │ líneas del │  │ agregar    │  │      —         │  │    —     │
  │                       │  │ pedido 5   │  │ línea      │  │                │  │          │
  └───────────────────────┘  └────────────┘  └────────────┘  └────────────────┘  └──────────┘
```

Convenciones:

* **Plural y en minúsculas:** `/api/pedidos`, no `/api/Pedido` ni `/api/getPedidos`.
* **Sin verbos en la ruta:** el verbo ya es el método HTTP.
* **Jerarquía cuando hay pertenencia:** `/api/pedidos/5/lineas`. No anides más de uno o dos niveles.
* **Filtros, orden y paginación por query string:** `/api/pedidos?estado=pendiente&page=2&pageSize=20`.

**Excepción razonable:** las operaciones del negocio que no son CRUD. `POST /api/pedidos/5/cancelar` es más claro que un `PATCH` con `{ "estado": "Cancelado" }` que esconde las reglas de cancelación. Es un sub-recurso "acción", y es una práctica aceptada (Google y Microsoft la usan en sus guías).

### 2. Propiedades de los métodos

| Método | Para qué | Seguro (no modifica) | Idempotente | Cuerpo en la petición |
| --- | --- | --- | --- | --- |
| `GET` | Leer | Sí | Sí | No |
| `POST` | Crear o ejecutar una acción | No | **No** | Sí |
| `PUT` | Reemplazar el recurso completo | No | Sí | Sí |
| `PATCH` | Modificar parte del recurso | No | No necesariamente | Sí |
| `DELETE` | Eliminar | No | Sí | No |

¿Por qué importa la idempotencia? Porque las redes fallan. Si un cliente envía `PUT /pedidos/5` y se corta la conexión antes de la respuesta, puede **reintentar sin miedo**: el resultado es el mismo. Con `POST /pedidos`, reintentar puede crear **dos** pedidos. (Para eso existen las *idempotency keys*, en [Para profundizar](#para-profundizar).)

### 3. Códigos de estado: el cliente decide por el número

| Código | Nombre | Cuándo |
| --- | --- | --- |
| **200** | OK | GET exitoso, o PUT/PATCH que devuelve el recurso |
| **201** | Created | POST que creó un recurso. Incluye el header `Location` con su URL |
| **204** | No Content | DELETE o PUT exitoso sin cuerpo |
| **400** | Bad Request | Datos mal formados o inválidos |
| **401** | Unauthorized | No se identificó (falta el token o no es válido) |
| **403** | Forbidden | Se identificó, pero no tiene permiso |
| **404** | Not Found | El recurso no existe |
| **409** | Conflict | Choca con el estado actual (email duplicado, pedido ya cancelado) |
| **422** | Unprocessable Content | Bien formado pero viola una regla del negocio (alternativa a 400/409) |
| **500** | Internal Server Error | Error inesperado del servidor (un bug) |

La regla mental: **4xx = el cliente puede corregir algo; 5xx = el problema es del servidor.**

### 4. En ASP.NET Core: controladores con `[ApiController]`

```csharp
[ApiController]
[Route("api/pedidos")]
public class PedidosController(PedidoStore store) : ControllerBase
{
    [HttpGet("{id:int}")]
    public ActionResult<PedidoDto> Obtener(int id)
    {
        var pedido = store.Buscar(id);
        if (pedido is null) return NotFound();
        return pedido;              // 200 OK con el JSON
    }

    [HttpPost]
    public ActionResult<PedidoDto> Crear(CrearPedidoRequest request)
    {
        var pedido = store.Crear(request);
        return CreatedAtAction(nameof(Obtener), new { id = pedido.Id }, pedido);   // 201 + Location
    }
}
```

* **`[ApiController]`:** valida el modelo automáticamente (si `request` es inválido, responde 400 sin entrar al método), infiere de dónde vienen los parámetros (`[FromBody]`, `[FromRoute]`) y formatea los errores como ProblemDetails.
* **`ActionResult<T>`:** permite devolver el DTO (200) o un resultado (`NotFound()`, `BadRequest()`), y documenta el tipo para OpenAPI.
* **`{id:int}`:** restricción de ruta. `/api/pedidos/abc` no entra al método (404).
* **`CreatedAtAction`:** genera el `Location` a partir del nombre de la acción, sin escribir la URL a mano.

### 5. DTOs: el contrato no es tu modelo de dominio

Nunca devuelvas tus entidades de dominio o de base de datos directamente:

* Exponen campos internos (hashes, columnas de auditoría).
* Un cambio interno (renombrar una propiedad) **rompe el contrato** con los clientes.
* Pueden generar ciclos de serialización (pedido → cliente → pedidos...).

Usa records como DTOs: uno para lo que entra (`CrearPedidoRequest`) y otro para lo que sale (`PedidoDto`).

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web`:

```csharp
using System.Collections.Concurrent;
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddSingleton<PedidoStore>();

var app = builder.Build();
app.MapControllers();
app.Run();

public record CrearPedidoRequest(
    [Required, StringLength(100)] string Cliente,
    [Range(typeof(decimal), "0.01", "1000000")] decimal Total);

public record PedidoDto(int Id, string Cliente, decimal Total, string Estado);

public class PedidoStore
{
    private readonly ConcurrentDictionary<int, PedidoDto> _pedidos = new();
    private int _ultimoId;

    public IEnumerable<PedidoDto> Listar() => _pedidos.Values.OrderBy(p => p.Id);

    public PedidoDto? Buscar(int id) => _pedidos.GetValueOrDefault(id);

    public PedidoDto Crear(CrearPedidoRequest r)
    {
        var pedido = new PedidoDto(Interlocked.Increment(ref _ultimoId), r.Cliente, r.Total, "Pendiente");
        _pedidos[pedido.Id] = pedido;
        return pedido;
    }

    public bool Reemplazar(int id, CrearPedidoRequest r)
    {
        if (!_pedidos.TryGetValue(id, out var actual)) return false;
        _pedidos[id] = actual with { Cliente = r.Cliente, Total = r.Total };
        return true;
    }

    public bool Eliminar(int id) => _pedidos.TryRemove(id, out _);
}

[ApiController]
[Route("api/pedidos")]
public class PedidosController(PedidoStore store) : ControllerBase
{
    [HttpGet]
    public IEnumerable<PedidoDto> Listar() => store.Listar();

    [HttpGet("{id:int}")]
    public ActionResult<PedidoDto> Obtener(int id)
    {
        var pedido = store.Buscar(id);
        if (pedido is null) return NotFound();
        return pedido;
    }

    [HttpPost]
    public ActionResult<PedidoDto> Crear(CrearPedidoRequest request)
    {
        var pedido = store.Crear(request);
        return CreatedAtAction(nameof(Obtener), new { id = pedido.Id }, pedido);
    }

    [HttpPut("{id:int}")]
    public IActionResult Reemplazar(int id, CrearPedidoRequest request) =>
        store.Reemplazar(id, request) ? NoContent() : NotFound();

    [HttpDelete("{id:int}")]
    public IActionResult Eliminar(int id) =>
        store.Eliminar(id) ? NoContent() : NotFound();
}
```

Pruébalo con un archivo `.http` (Visual Studio y VS Code lo ejecutan; ajusta el puerto al de `launchSettings.json`):

```http
@base = http://localhost:5000

POST {{base}}/api/pedidos
Content-Type: application/json

{ "cliente": "Ana", "total": 150.50 }

###
GET {{base}}/api/pedidos/1

###
POST {{base}}/api/pedidos
Content-Type: application/json

{ "cliente": "", "total": -5 }
```

Respuestas:

```text
POST válido   → 201 Created
                Location: http://localhost:5000/api/pedidos/1
                { "id": 1, "cliente": "Ana", "total": 150.50, "estado": "Pendiente" }

GET 1         → 200 OK    { "id": 1, ... }
GET 99        → 404 Not Found
POST inválido → 400 Bad Request (application/problem+json)
                { "title": "One or more validation errors occurred.", "status": 400,
                  "errors": { "Cliente": [...], "Total": [...] }, "traceId": "..." }
```

El 400 lo generó `[ApiController]` sin una sola línea de validación en el método. Ese formato es **ProblemDetails**: lo ves en detalle en la [lección siguiente](02-Errores%20con%20ProblemDetails.md).

-----

## Errores comunes

**1. Verbos en las rutas (`/getPedidos`, `/crearPedido`).**
Qué pasa: rutas inconsistentes, imposibles de adivinar.
Por qué: se piensa en "funciones remotas", no en recursos.
Arreglo: sustantivos en plural y el método HTTP como verbo.

**2. GET que modifica datos.**
Qué pasa: un *crawler*, un *prefetch* o un reintento automático ejecutan cambios.
Por qué: GET se considera seguro; cachés y navegadores lo repiten libremente.
Arreglo: GET solo lee; los cambios, con POST/PUT/PATCH/DELETE.

**3. `200 OK` con un error en el cuerpo.**
Qué pasa: `HttpClient.EnsureSuccessStatusCode()`, el monitoreo y los reintentos creen que todo salió bien.
Por qué: se usa el cuerpo para comunicar el resultado en lugar del código.
Arreglo: el código de estado correcto (4xx/5xx) con un cuerpo ProblemDetails.

**4. POST que devuelve 200 sin `Location`.**
Qué pasa: el cliente no sabe dónde quedó el recurso creado.
Por qué: se usa `Ok(...)` para todo.
Arreglo: `CreatedAtAction(nameof(Obtener), new { id }, dto)`.

**5. Devolver entidades de dominio o de EF.**
Qué pasa: fugas de datos internos, ciclos de serialización y contratos que se rompen con cualquier refactor.
Por qué: es lo más rápido de escribir.
Arreglo: DTOs (records) de entrada y de salida.

**6. "No route matches the supplied values" en `CreatedAtAction`.**
Qué pasa: `InvalidOperationException` al generar el `Location`.
Por qué: el nombre de la acción o los valores de ruta no coinciden con ningún endpoint (por ejemplo, falta `id` o, con versionamiento por URL, falta `version`).
Arreglo: `nameof(Accion)` y todos los parámetros de la ruta en el objeto anónimo.

-----

## Según la versión de .NET

* **ASP.NET Core 2.1:** `[ApiController]` y `ActionResult<T>`.
* **ASP.NET Core 2.2:** ProblemDetails en las respuestas de error automáticas.
* **.NET 6:** *minimal APIs* (`app.MapGet(...)`), alternativa a los controladores.
* **.NET 8:** `IExceptionHandler`; generación de OpenAPI más integrada.
* **.NET 9:** `Microsoft.AspNetCore.OpenApi` reemplaza a Swashbuckle en las plantillas.
* **C# 12:** constructores primarios, como `PedidosController(PedidoStore store)`.

-----

## Cuándo sí y cuándo no

**REST sobre HTTP encaja cuando:**

* Expones recursos a clientes diversos (web, móvil, terceros).
* Quieres aprovechar la infraestructura HTTP: cachés, proxies, códigos de estado, herramientas.

**Considera otra cosa cuando:**

* La comunicación es entre tus propios servicios con alto rendimiento: **gRPC**.
* El cliente necesita armar consultas muy variables sobre un grafo de datos: **GraphQL**.
* Necesitas notificaciones en tiempo real: **SignalR / WebSockets**.

-----

## Resumen en 5 líneas

1. Recursos con sustantivos en plural; el verbo es el método HTTP.
2. GET es seguro; PUT y DELETE son idempotentes; POST no.
3. El código de estado comunica el resultado: 2xx éxito, 4xx error del cliente, 5xx error del servidor.
4. POST de creación → 201 con `Location` (`CreatedAtAction`).
5. DTOs como contrato; nunca expongas las entidades internas.

-----

## Para profundizar

<details>
<summary>Idempotency keys: reintentar un POST sin duplicar</summary>

El cliente genera un identificador único por operación y lo envía en un header (`Idempotency-Key: 6f1c...`). El servidor guarda la clave con la respuesta; si llega otra petición con la misma clave, devuelve la respuesta guardada en lugar de crear otro recurso. Es lo que usan las APIs de pagos (Stripe, por ejemplo) para que un reintento no cobre dos veces.

</details>

<details>
<summary>El modelo de madurez de Richardson</summary>

Leonard Richardson clasificó las APIs en niveles:

* **Nivel 0:** un único endpoint y todo por POST (estilo RPC/SOAP).
* **Nivel 1:** recursos con URIs propias.
* **Nivel 2:** métodos HTTP y códigos de estado usados correctamente. **La mayoría de las APIs "REST" del mundo real están aquí.**
* **Nivel 3:** hipermedia (HATEOAS): las respuestas incluyen enlaces a las acciones posibles. Lo ves en [HATEOAS](04-HATEOAS.md).

</details>

<details>
<summary>Controladores o minimal APIs</summary>

Lo mismo con *minimal APIs*:

```csharp
app.MapGet("/api/pedidos/{id:int}", Results<Ok<PedidoDto>, NotFound> (int id, PedidoStore store) =>
    store.Buscar(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound());
```

Son más livianas y rápidas de arrancar; los controladores agrupan mejor muchas acciones y tienen filtros y convenciones maduras. Ambas son válidas en .NET 10; este módulo usa controladores porque es lo que usan las notas de clase y la mayoría de los proyectos existentes.

</details>

-----

## En entrevista

### Respuesta corta (junior)

REST es un estilo para diseñar APIs sobre HTTP donde todo es un recurso con su URL, como `/api/pedidos/5`, y las acciones se expresan con los métodos HTTP: GET para leer, POST para crear, PUT para reemplazar y DELETE para borrar. La respuesta usa el código de estado correcto: 200 si salió bien, 201 si se creó, 404 si no existe, 400 si los datos están mal.

### Respuesta ampliada (semi-senior)

Diseño la API como un contrato: recursos en plural, métodos HTTP según su semántica (GET seguro, PUT y DELETE idempotentes, POST no idempotente, por eso uso idempotency keys en operaciones críticas), y códigos de estado precisos, con 201 + `Location` en las creaciones, 409 o 422 para conflictos de negocio y errores en formato ProblemDetails. Expongo DTOs, no entidades, para que el modelo interno pueda evolucionar. En ASP.NET Core uso `[ApiController]` para la validación automática y `ActionResult<T>` para documentar las respuestas en OpenAPI. Para operaciones del negocio que no son CRUD, uso sub-recursos de acción (`POST /pedidos/5/cancelar`).

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre PUT y PATCH?**
PUT reemplaza el recurso completo y es idempotente; PATCH modifica solo algunos campos y no es necesariamente idempotente.

**2. ¿401 o 403?**
401: no sé quién eres (sin autenticación válida). 403: sé quién eres, pero no tienes permiso.

**3. ¿Por qué no devolver siempre 200 con un campo `success`?**
Porque las herramientas HTTP (clientes, cachés, monitoreo, reintentos) se basan en el código de estado; el cuerpo es para los detalles.

-----

## Práctica

**Ejercicio 1.** Rediseña estas rutas al estilo REST (método + ruta):

```text
POST /api/obtenerClientes
POST /api/cliente/nuevo
GET  /api/eliminarCliente?id=3
POST /api/clientes/3/actualizarEmail
GET  /api/pedidosDeCliente?clienteId=3
```

<details>
<summary>Solución</summary>

```text
GET    /api/clientes
POST   /api/clientes
DELETE /api/clientes/3
PATCH  /api/clientes/3            { "email": "..." }
GET    /api/clientes/3/pedidos
```

</details>

**Ejercicio 2.** ¿Qué código de estado devuelves en cada caso?

1. Se creó un cliente.
2. Se pide el cliente 99, que no existe.
3. Se intenta registrar un email que ya existe.
4. Se borró un pedido.
5. El token expiró.
6. Una consulta a la base de datos lanzó una excepción no controlada.

<details>
<summary>Solución</summary>

1. **201 Created**, con `Location`.
2. **404 Not Found.**
3. **409 Conflict** (choca con el estado actual).
4. **204 No Content.**
5. **401 Unauthorized.**
6. **500 Internal Server Error**, sin detalles internos en la respuesta (solo en los logs).

</details>

-----

## Siguiente lección

[Errores con ProblemDetails](02-Errores%20con%20ProblemDetails.md)
