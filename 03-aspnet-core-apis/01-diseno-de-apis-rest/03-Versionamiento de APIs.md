# Versionamiento de APIs

## En una frase

**Versionar** una API es publicar los cambios que rompen el contrato como una **versión nueva** (v2) mientras la anterior (v1) sigue funcionando, para que los clientes existentes no se rompan y puedan migrar a su ritmo.

-----

## Antes de empezar

Conviene que ya sepas:

* Rutas, DTOs y `[ApiController]`: [Principios REST y códigos de estado](01-Principios%20REST%20y%20codigos%20de%20estado.md).
* ProblemDetails: [Errores con ProblemDetails](02-Errores%20con%20ProblemDetails.md).
* Instalar paquetes NuGet: `dotnet add package Asp.Versioning.Mvc`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Contrato:** todo lo que un cliente asume de tu API: rutas, campos, tipos, códigos de estado y comportamiento.
* **Cambio que rompe (*breaking change*):** cambio que hace fallar a un cliente que funcionaba.
* **Versión mayor / menor:** `2.0` indica cambios que rompen; `1.1`, cambios compatibles (si versionas las menores).
* **Deprecación:** anunciar que una versión dejará de existir, sin quitarla todavía.
* **Sunset:** la fecha en que una versión deprecada se apaga.
* **Lector tolerante (*tolerant reader*):** cliente que ignora los campos que no conoce, en lugar de fallar.

-----

## El problema

Tu API tiene una app móvil que la consume:

```json
GET /api/pedidos/1
{ "id": 1, "cliente": "Ana", "total": 150.0 }
```

El negocio pide manejar varias monedas. Cambias la respuesta:

```json
{ "id": 1, "cliente": { "id": 7, "nombre": "Ana" }, "total": { "monto": 150.0, "moneda": "PEN" } }
```

Al desplegar, **la app móvil se cae**: esperaba un texto en `cliente` y un número en `total`. Los usuarios tienen la versión vieja de la app instalada y no todos actualizan. Tienes que soportar **los dos formatos a la vez**, durante meses.

-----

## Cómo funciona

### 1. Primero: ¿realmente necesitas una versión nueva?

Versionar tiene un costo: mantener, probar y documentar dos contratos. Muchos cambios **no** lo requieren:

| Cambio | ¿Rompe? |
| --- | --- |
| Agregar un campo nuevo a la respuesta | **No** (si los clientes son lectores tolerantes) |
| Agregar un endpoint nuevo | No |
| Agregar un parámetro **opcional** a la petición | No |
| Quitar o renombrar un campo | **Sí** |
| Cambiar el tipo de un campo (`string` → objeto, `int` → `string`) | **Sí** |
| Hacer obligatorio un campo que era opcional | **Sí** |
| Cambiar una ruta o un código de estado | **Sí** |
| Cambiar el significado de un campo (`total` con o sin impuestos) | **Sí**, aunque el JSON sea igual |

Regla: **evoluciona agregando; versiona solo cuando tienes que quitar o cambiar.**

### 2. Las estrategias

```text
 URL segment      GET /api/v2/pedidos/1
 Query string     GET /api/pedidos/1?api-version=2.0
 Header           GET /api/pedidos/1          api-version: 2.0
 Media type       GET /api/pedidos/1          Accept: application/json;v=2.0
```

| Estrategia | Ventajas | Desventajas | Quién la usa |
| --- | --- | --- | --- |
| **URL** | Visible, fácil de probar en el navegador, cachea bien, simple de enrutar | La URL de un recurso cambia entre versiones (en REST puro, el recurso es el mismo) | La mayoría de las APIs públicas |
| **Query string** | Visible y la ruta no cambia | Fácil de olvidar; requiere un valor por defecto | Azure (`?api-version=2024-01-01`) |
| **Header** | URLs limpias y estables | Invisible en el navegador; las cachés deben variar por el header (`Vary`) | GitHub (`X-GitHub-Api-Version`), Stripe (versiones por fecha) |
| **Media type** | El más "REST puro" | El más complejo para clientes y herramientas | Poco frecuente |

No hay una ganadora universal: no depende de si el proyecto es "simple" o "empresarial", sino de quiénes son tus clientes y de tu infraestructura. **URL** es la opción por defecto razonable por su simplicidad; lo importante es **elegir una y ser consistente**.

### 3. En ASP.NET Core: `Asp.Versioning.Mvc`

El paquete oficial es **`Asp.Versioning.Mvc`** (para controladores) o **`Asp.Versioning.Http`** (para minimal APIs). El antiguo `Microsoft.AspNetCore.Mvc.Versioning` está deprecado.

```csharp
builder.Services
    .AddApiVersioning(opciones =>
    {
        opciones.DefaultApiVersion = new ApiVersion(1, 0);
        opciones.ReportApiVersions = true;                            // headers api-supported-versions
        opciones.ApiVersionReader = new UrlSegmentApiVersionReader(); // lee la versión de la URL
    })
    .AddMvc();                                                        // integración con controladores
```

Sin esta configuración, los atributos `[ApiVersion]` no hacen nada (o la aplicación falla al resolver la ruta `{version:apiVersion}`, porque la restricción `apiVersion` no existe).

### 4. Dos versiones en un mismo controlador: `[MapToApiVersion]`

```csharp
[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/pedidos")]
public class PedidosController : ControllerBase
{
    [HttpGet("{id:int}")]
    [MapToApiVersion("1.0")]
    public PedidoV1 ObtenerV1(int id) => ...;

    [HttpGet("{id:int}")]
    [MapToApiVersion("2.0")]
    public PedidoV2 ObtenerV2(int id) => ...;
}
```

* El **controlador** declara qué versiones soporta (`[ApiVersion]`).
* Cada **acción** indica a cuál pertenece (`[MapToApiVersion]`). Las acciones sin `MapToApiVersion` responden en **todas** las versiones del controlador.
* Dos acciones con la misma ruta sin `MapToApiVersion` → `AmbiguousMatchException` en tiempo de ejecución.

### 5. O un controlador por versión

Cuando las versiones difieren mucho, sepáralas:

```text
Controllers/
  V1/PedidosController.cs   [ApiVersion("1.0")]  [Route("api/v{version:apiVersion}/pedidos")]
  V2/PedidosController.cs   [ApiVersion("2.0")]  [Route("api/v{version:apiVersion}/pedidos")]
```

Cada controlador es simple; la lógica del negocio sigue compartida en la capa de aplicación. **Solo cambian los DTOs y el mapeo**, nunca el dominio.

### 6. Deprecar y apagar

```text
 v1 activa ──► v2 publicada ──► v1 deprecada ──► aviso de sunset ──► v1 apagada
                                (Deprecated = true:  header api-deprecated-versions: 1.0)
```

* `Deprecated = true` + `ReportApiVersions` hace que cada respuesta incluya `api-supported-versions: 2.0` y `api-deprecated-versions: 1.0`.
* Comunica la fecha de apagado (changelog, correo, header `Sunset` del RFC 8594, que `Asp.Versioning` puede emitir con *sunset policies*).
* Mide quién sigue usando v1 (logs por versión) antes de apagarla.

-----

## Ejemplo completo

`Program.cs` de un proyecto `dotnet new web` con el paquete `Asp.Versioning.Mvc`:

```csharp
using Asp.Versioning;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services
    .AddApiVersioning(opciones =>
    {
        opciones.DefaultApiVersion = new ApiVersion(1, 0);
        opciones.ReportApiVersions = true;
        opciones.ApiVersionReader = new UrlSegmentApiVersionReader();
    })
    .AddMvc();

var app = builder.Build();
app.MapControllers();
app.Run();

// v1: el contrato original
public record PedidoV1(int Id, string Cliente, decimal Total);

// v2: cliente como objeto y total con moneda (cambios que rompen)
public record PedidoV2(int Id, ClienteResumen Cliente, Dinero Total);
public record ClienteResumen(int Id, string Nombre);
public record Dinero(decimal Monto, string Moneda);

[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/pedidos")]
public class PedidosController : ControllerBase
{
    [HttpGet("{id:int}")]
    [MapToApiVersion("1.0")]
    public PedidoV1 ObtenerV1(int id) => new(id, "Ana", 150m);

    [HttpGet("{id:int}")]
    [MapToApiVersion("2.0")]
    public PedidoV2 ObtenerV2(int id) =>
        new(id, new ClienteResumen(7, "Ana"), new Dinero(150m, "PEN"));

    [HttpGet("{id:int}/estado")]   // sin MapToApiVersion: existe en v1 y en v2
    public object Estado(int id) => new { id, estado = "Pendiente" };
}
```

Respuestas:

```text
GET /api/v1/pedidos/1
  200  api-supported-versions: 2.0
       api-deprecated-versions: 1.0
       { "id": 1, "cliente": "Ana", "total": 150 }

GET /api/v2/pedidos/1
  200  { "id": 1, "cliente": { "id": 7, "nombre": "Ana" }, "total": { "monto": 150, "moneda": "PEN" } }

GET /api/v2/pedidos/1/estado   → 200 (la acción existe en las dos versiones)
GET /api/v3/pedidos/1          → error ProblemDetails: versión no soportada
```

La app móvil vieja sigue usando `/api/v1` sin enterarse; la nueva usa `/api/v2`.

-----

## Errores comunes

**1. `[ApiVersion]` sin `AddApiVersioning()`.**
Qué pasa: la restricción `{version:apiVersion}` no se reconoce (`InvalidOperationException` al construir las rutas) o los atributos se ignoran.
Por qué: los atributos solo son metadatos; el paquete los interpreta.
Arreglo: `AddApiVersioning(...).AddMvc()`.

**2. Poner `[ApiVersion("2.0")]` en una acción suelta para "crear la v2".**
Qué pasa: en la guía de clase, `GetV2` no tenía `[HttpGet("{id}")]`, así que respondía en la ruta del controlador con cualquier método HTTP y el `id` venía de la query string. Y si se le agrega la misma ruta que la v1 sin asignar versiones, las dos acciones chocan (`AmbiguousMatchException`).
Por qué: la forma documentada es que el **controlador** declare sus versiones con `[ApiVersion]` y que cada **acción** se asigne con `[MapToApiVersion]`.
Arreglo: `[ApiVersion("1.0")]` y `[ApiVersion("2.0")]` en el controlador; en cada acción, el atributo HTTP con su ruta y `[MapToApiVersion("x")]`.

**3. Crear una versión nueva para agregar un campo.**
Qué pasa: dos contratos que mantener sin necesidad.
Por qué: se confunde "cambio" con "cambio que rompe".
Arreglo: agregar campos opcionales es compatible; versiona solo lo que rompe.

**4. Duplicar el dominio por versión.**
Qué pasa: bugs que se corrigen en v2 pero no en v1.
Por qué: se copió todo el proyecto o la lógica dentro de cada controlador.
Arreglo: las versiones difieren en DTOs y mapeo; el dominio y los casos de uso son únicos.

**5. Enlaces o `CreatedAtAction` sin la versión.**
Qué pasa: "No route matches the supplied values" o enlaces que apuntan a otra versión.
Por qué: con versionamiento por URL, `version` es un parámetro de ruta más.
Arreglo: incluye `version = RouteData.Values["version"]` en los valores de ruta (toma el valor tal como vino en la URL, por ejemplo `1`).

**6. Mezclar estrategias sin criterio.**
Qué pasa: unos endpoints versionan por URL y otros por header.
Por qué: cada desarrollador eligió una.
Arreglo: una estrategia para toda la API (o `ApiVersionReader.Combine(...)` de forma deliberada, documentada).

-----

## Según la versión de .NET

* **ASP.NET Core 2.x–5:** `Microsoft.AspNetCore.Mvc.Versioning` (hoy deprecado).
* **.NET 6+ (Asp.Versioning 6):** el proyecto pasó a la .NET Foundation como `Asp.Versioning.*`; soporte para minimal APIs (`Asp.Versioning.Http`) y respuestas de error con ProblemDetails.
* **Asp.Versioning 8:** para .NET 8 y posteriores; *sunset policies* y mejor integración con OpenAPI (`Asp.Versioning.Mvc.ApiExplorer`).

-----

## Cuándo sí y cuándo no

**Versiona cuando:**

* Tienes clientes que no controlas o que no actualizan al mismo tiempo (apps móviles, terceros, integraciones).
* Necesitas un cambio que rompe el contrato.

**No hace falta (todavía) cuando:**

* El único cliente es tu propio frontend y se despliega junto con la API.
* Los cambios son aditivos.

Aun así, en una API nueva conviene **dejar preparada la v1** desde el primer día: agregarla después obliga a cambiar todas las URLs.

-----

## Resumen en 5 líneas

1. Versiona solo los cambios que rompen; los cambios aditivos no necesitan versión nueva.
2. Estrategias: URL (la más común), query string, header o media type; elige una y sé consistente.
3. En ASP.NET Core: `Asp.Versioning.Mvc` con `AddApiVersioning().AddMvc()`.
4. El controlador declara versiones con `[ApiVersion]`; las acciones se asignan con `[MapToApiVersion]`.
5. Depreca con aviso (`Deprecated = true`, headers, fecha de sunset) y mide el uso antes de apagar.

-----

## Para profundizar

<details>
<summary>Aceptar varias formas de indicar la versión</summary>

```csharp
opciones.ApiVersionReader = ApiVersionReader.Combine(
    new QueryStringApiVersionReader("api-version"),
    new HeaderApiVersionReader("api-version"));
opciones.AssumeDefaultVersionWhenUnspecified = true;   // sin versión → DefaultApiVersion
```

`AssumeDefaultVersionWhenUnspecified` no aplica al versionamiento por URL: si la ruta exige `{version}`, una URL sin versión simplemente no coincide.

</details>

<details>
<summary>OpenAPI con varias versiones</summary>

Con `Asp.Versioning.Mvc.ApiExplorer` (`.AddApiExplorer(o => { o.GroupNameFormat = "'v'VVV"; o.SubstituteApiVersionInUrl = true; })`) cada versión aparece como un documento OpenAPI separado (`v1`, `v2`), con la versión ya sustituida en las rutas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Versionar una API es tener, por ejemplo, `/api/v1/pedidos` y `/api/v2/pedidos` al mismo tiempo, para que los clientes viejos sigan funcionando cuando cambias el formato de las respuestas. En ASP.NET Core se hace con el paquete `Asp.Versioning`, usando `[ApiVersion]` en el controlador y `[MapToApiVersion]` en las acciones.

### Respuesta ampliada (semi-senior)

Primero evalúo si el cambio rompe el contrato: los cambios aditivos los publico sin versión nueva, porque espero clientes tolerantes. Si rompe, publico una versión mayor. La estrategia depende de los clientes: URL por simplicidad y cacheo, header o query string si quiero URLs estables. En ASP.NET Core uso `Asp.Versioning.Mvc`, declaro versiones por controlador y asigno acciones con `MapToApiVersion`, o separo controladores por versión si divergen mucho; el dominio es único y solo cambian DTOs y mapeos. Activo `ReportApiVersions`, marco versiones deprecadas, comunico el sunset y mido el tráfico por versión antes de apagar una.

### Preguntas frecuentes de seguimiento

**1. ¿Agregar un campo a la respuesta requiere una versión nueva?**
No, si los clientes ignoran campos desconocidos. Quitar, renombrar o cambiar el tipo sí.

**2. ¿Por qué no simplemente actualizar a todos los clientes?**
Porque no siempre los controlas: apps móviles instaladas, integraciones de terceros, equipos con otros calendarios.

**3. ¿Cuántas versiones mantienes en paralelo?**
Pocas, normalmente la actual y la anterior, con una política de deprecación y fechas comunicadas.

-----

## Práctica

**Ejercicio 1.** Clasifica cada cambio: ¿rompe o no rompe?

1. Agregar `fechaCreacion` a la respuesta de `GET /pedidos/{id}`.
2. Renombrar `cliente` a `nombreCliente`.
3. Que `POST /pedidos` exija un campo nuevo `direccionEnvio`.
4. Agregar `GET /pedidos/{id}/historial`.
5. Que `DELETE /pedidos/{id}` pase de devolver 200 con el pedido a 204 sin cuerpo.
6. Que `total` pase a incluir impuestos (el JSON no cambia).

<details>
<summary>Solución</summary>

1. No rompe (aditivo).
2. **Rompe.**
3. **Rompe** (un cliente que no lo envía ahora recibe 400).
4. No rompe.
5. **Rompe** (el cliente que leía el cuerpo falla).
6. **Rompe** el significado: los clientes calcularán mal. Es el más peligroso porque no se nota.

</details>

**Ejercicio 2.** Tienes una v1 con `GET /api/v1/clientes/{id}` que devuelve `{ "id", "nombre" }`. Crea la v2, donde `nombre` se separa en `nombres` y `apellidos`, sin tocar la v1.

<details>
<summary>Solución</summary>

```csharp
public record ClienteV1(int Id, string Nombre);
public record ClienteV2(int Id, string Nombres, string Apellidos);

[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/clientes")]
public class ClientesController : ControllerBase
{
    private static readonly (string Nombres, string Apellidos) Ana = ("Ana María", "Pérez Soto");

    [HttpGet("{id:int}")]
    [MapToApiVersion("1.0")]
    public ClienteV1 ObtenerV1(int id) => new(id, $"{Ana.Nombres} {Ana.Apellidos}");

    [HttpGet("{id:int}")]
    [MapToApiVersion("2.0")]
    public ClienteV2 ObtenerV2(int id) => new(id, Ana.Nombres, Ana.Apellidos);
}
```

La v1 se sigue armando a partir de los mismos datos: el modelo interno es uno solo; cada versión es un mapeo distinto.

</details>

-----

## Siguiente lección

[HATEOAS](04-HATEOAS.md)
