# Documentación con OpenAPI

## En una frase

OpenAPI es un formato estándar (JSON o YAML) que describe rutas, parámetros, cuerpos y respuestas de una API; desde .NET 9, ASP.NET Core lo genera con `Microsoft.AspNetCore.OpenApi`, y una interfaz como Swagger UI o Scalar solo **muestra** ese documento, cuya calidad depende de los tipos y metadatos de tus endpoints.

-----

## Antes de empezar

Conviene que ya sepas:

* Tipos de retorno en controladores: [Controladores y rutas por atributo](01-Controladores%20y%20rutas%20por%20atributo.md).
* `TypedResults` y `Results<...>` en Minimal APIs: [Typed Results](../02-minimal-apis/02-Typed%20Results.md).
* Entornos y `launchSettings.json`: [Host, configuración y pipeline HTTP](../00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **OpenAPI:** especificación estándar para describir APIs HTTP; antes se llamaba Swagger Specification.
* **Swagger:** nombre de un conjunto de herramientas (Swagger UI, Swagger Editor); hoy se usa coloquialmente para todo lo de OpenAPI.
* **Documento OpenAPI:** archivo JSON o YAML con la descripción completa de una API.
* **Swagger UI / Scalar:** interfaces web que leen un documento OpenAPI y permiten explorar y probar la API.
* **Esquema (*schema*):** descripción de la forma de un JSON (propiedades, tipos, obligatoriedad).

-----

## El problema

Un equipo sigue un tutorial de 2022 sobre un proyecto .NET 10:

```csharp
builder.Services.AddSwaggerGen();
// ...
app.UseSwagger();
app.UseSwaggerUI();
```

Y escribe los endpoints así:

```csharp
[HttpGet("{id}")]
public IActionResult Obtener(int id) => ...;

app.MapPost("/tareas", (CrearTareaRequest r) => Results.Ok(servicio.Crear(r)));
```

Lo que pasa:

1. `AddSwaggerGen` no compila: el proyecto no tiene `Swashbuckle.AspNetCore`, porque la plantilla de .NET 9 en adelante ya no lo incluye. Lo agregan, y ahora conviven dos generadores (el integrado y Swashbuckle) con configuraciones distintas.
2. Swagger UI muestra `Obtener` con respuesta `200` **sin esquema**: `IActionResult` no dice qué devuelve. El `404` no aparece.
3. El `POST` figura con `200` y sin cuerpo de respuesta: `Results.Ok(...)` devuelve `IResult`, que no lleva tipo.
4. El frontend genera su cliente TypeScript desde ese documento y obtiene `any` en todas las respuestas.
5. Alguien argumenta que "Swagger solo en Development es una medida de seguridad", mientras los endpoints de administración siguen sin autenticación.

Las consecuencias:

* documentación que no coincide con lo que la API realmente hace;
* clientes generados sin tipos;
* una falsa sensación de seguridad.

-----

## Cómo funciona

### 1. Las piezas, sin confundirlas

```text
Tus endpoints + sus tipos y metadatos
            │  (ApiExplorer / metadatos de endpoints)
            ▼
Generador del documento ── Microsoft.AspNetCore.OpenApi (integrado)  o  Swashbuckle / NSwag
            │
            ▼
Documento OpenAPI  /openapi/v1.json  (estándar, independiente de .NET)
            │
            ├──► Interfaz para humanos: Scalar, Swagger UI, Redoc
            ├──► Generadores de clientes: Kiota, NSwag, openapi-generator
            └──► Pruebas de contrato, gateways, catálogos de APIs
```

El documento es el producto. La interfaz es solo una forma de leerlo.

### 2. El generador integrado (.NET 9+)

```csharp
builder.Services.AddOpenApi();        // registra el generador

var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                 // expone /openapi/v1.json
}
```

`AddOpenApi` viene en el paquete `Microsoft.AspNetCore.OpenApi`, que la plantilla `webapi` ya incluye. No trae interfaz gráfica. Para tener una:

| Interfaz | Paquete | Registro |
| --- | --- | --- |
| Scalar | `Scalar.AspNetCore` | `app.MapScalarApiReference();` → `/scalar` |
| Swagger UI | `Swashbuckle.AspNetCore.SwaggerUI` | `app.UseSwaggerUI(o => o.SwaggerEndpoint("/openapi/v1.json", "v1"));` |

Swashbuckle (el generador completo) sigue existiendo como paquete de terceros. Lo que no conviene es mezclar dos generadores en el mismo proyecto.

### 3. La calidad del documento depende de tus tipos

| Endpoint | Lo que documenta |
| --- | --- |
| Controlador con `IActionResult` | `200` sin esquema |
| Controlador con `ActionResult<TareaDto>` | `200` con el esquema de `TareaDto` |
| + `[ProducesResponseType(404)]` | además el `404` |
| Minimal API con `Results.Ok(x)` (`IResult`) | `200` sin esquema |
| Minimal API con `Results<Ok<TareaDto>, NotFound>` | `200` con esquema y `404`, sin atributos |

```csharp
// Controlador
[HttpGet("{id:int}")]
[ProducesResponseType<TareaDto>(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public ActionResult<TareaDto> Obtener(int id) => ...;

// Minimal API: el tipo de retorno ya es la documentación
static Results<Ok<TareaDto>, NotFound> Obtener(int id, ITareaService s) => ...;
```

### 4. Descripciones y agrupación

```csharp
app.MapGroup("/tareas")
   .WithTags("Tareas");

grupo.MapGet("/{id:int}", Obtener)
     .WithName("ObtenerTarea")               // operationId: útil para clientes generados
     .WithSummary("Obtiene una tarea por id")
     .WithDescription("Devuelve 404 si la tarea no existe.");
```

En controladores, las mismas descripciones pueden salir de los comentarios XML (`/// <summary>`). En .NET 10, el generador integrado los lee si activas `<GenerateDocumentationFile>true</GenerateDocumentationFile>` en el `.csproj`.

### 5. "Solo en Development" no es seguridad

Exponer el documento solo en Development es una decisión razonable para APIs **internas**: reduce información disponible y superficie. Pero no protege nada por sí mismo. Un atacante puede descubrir los endpoints igual, observando el tráfico del frontend. La seguridad está en la autenticación y autorización de cada endpoint ([Seguridad de APIs](../03-seguridad-de-apis/README.md)).

Para APIs **públicas** o consumidas por otros equipos, publicar el documento es lo normal: es el contrato.

### 6. Que el navegador abra la documentación

En `Properties/launchSettings.json`:

```json
"launchBrowser": true,
"launchUrl": "scalar"
```

Solo afecta a `dotnet run` y al IDE en tu máquina; no se publica.

### 7. El documento como artefacto del build

Con el paquete `Microsoft.Extensions.ApiDescription.Server`, el documento se genera **al compilar** en un archivo. Así puedes versionarlo, revisarlo en un *pull request* ("¿este cambio rompe el contrato?") y generar clientes en el pipeline de CI sin levantar la aplicación.

-----

## Ejemplo completo

Proyecto `dotnet new web` con el paquete `Microsoft.AspNetCore.OpenApi` 10.0.0. `Program.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();   // GET /openapi/v1.json
}

var tareas = new List<TareaDto> { new(1, "Pagar la luz", true), new(2, "Revisar PR", false) };

var grupo = app.MapGroup("/tareas").WithTags("Tareas");

grupo.MapGet("/", () => TypedResults.Ok(tareas))
     .WithName("ListarTareas")
     .WithSummary("Lista todas las tareas");

grupo.MapGet("/{id:int}", Results<Ok<TareaDto>, NotFound> (int id) =>
        tareas.FirstOrDefault(t => t.Id == id) is { } tarea
            ? TypedResults.Ok(tarea)
            : TypedResults.NotFound())
     .WithName("ObtenerTarea")
     .WithSummary("Obtiene una tarea por id")
     .WithDescription("Devuelve 404 si la tarea no existe.");

grupo.MapPost("/", Created<TareaDto> (CrearTareaRequest request) =>
        {
            var tarea = new TareaDto(tareas.Max(t => t.Id) + 1, request.Titulo, false);
            tareas.Add(tarea);   // solo para el ejemplo: List<T> no es seguro entre hilos
            return TypedResults.Created($"/tareas/{tarea.Id}", tarea);
        })
     .WithName("CrearTarea")
     .WithSummary("Crea una tarea");

app.Run();

public sealed record TareaDto(int Id, string Titulo, bool Completada);

public sealed record CrearTareaRequest([Required, MaxLength(100)] string Titulo);
```

Fragmento de `GET /openapi/v1.json` (abreviado; el orden de las propiedades puede variar):

```json
{
  "openapi": "3.1.1",
  "info": { "title": "Tareas | v1", "version": "1.0.0" },
  "paths": {
    "/tareas/{id}": {
      "get": {
        "tags": [ "Tareas" ],
        "summary": "Obtiene una tarea por id",
        "description": "Devuelve 404 si la tarea no existe.",
        "operationId": "ObtenerTarea",
        "parameters": [
          { "name": "id", "in": "path", "required": true, "schema": { "type": "integer", "format": "int32" } }
        ],
        "responses": {
          "200": {
            "description": "OK",
            "content": { "application/json": { "schema": { "$ref": "#/components/schemas/TareaDto" } } }
          },
          "404": { "description": "Not Found" }
        }
      }
    }
  }
}
```

(El título sale del nombre del proyecto; aquí, un proyecto llamado `Tareas`.)

Qué observar:

* El `404` y el esquema de `TareaDto` aparecen sin un solo atributo: los tomó del tipo `Results<Ok<TareaDto>, NotFound>`.
* `WithName` se convirtió en `operationId`, y un generador de clientes producirá un método `ObtenerTarea`.
* El parámetro `id` figura como entero obligatorio por la restricción `{id:int}`.
* El documento solo se expone en Development por decisión de este ejemplo; la API sigue necesitando autenticación si no es pública.
* No hay interfaz gráfica: agrega Scalar o Swagger UI apuntando a `/openapi/v1.json` si la quieres.

-----

## Errores comunes

**1. Seguir tutoriales de Swashbuckle en .NET 9 o posterior.**
Qué pasa: `AddSwaggerGen` no existe, o terminas con dos generadores en paralelo.
Por qué: la plantilla cambió a `Microsoft.AspNetCore.OpenApi`.
Arreglo: `AddOpenApi()` + `MapOpenApi()`, y una interfaz que lea ese documento. Si eliges Swashbuckle, úsalo solo a él.

**2. Devolver `IActionResult` o `IResult` sin metadatos.**
Qué pasa: respuestas sin esquema y clientes generados con `any`/`object`.
Por qué: el generador no puede adivinar qué hay dentro.
Arreglo: `ActionResult<T>` o `[ProducesResponseType<T>]`; en Minimal APIs, `TypedResults` y `Results<...>`.

**3. Creer que ocultar la documentación protege la API.**
Qué pasa: endpoints sin autorización accesibles para quien los descubra.
Por qué: el documento describe la API; no la protege.
Arreglo: autenticación y autorización en cada endpoint; la visibilidad del documento es una decisión aparte.

**4. Confundir Swagger con OpenAPI.**
Qué pasa: discusiones de "migrar de Swagger a OpenAPI" que en realidad son cambiar de herramienta.
Por qué: OpenAPI es la especificación; Swagger, una familia de herramientas.
Arreglo: habla del documento (OpenAPI) y de la interfaz (Swagger UI, Scalar) por separado.

**5. No poner `operationId`.**
Qué pasa: clientes generados con métodos como `TareasGet2`.
Por qué: el generador inventa nombres a partir de la ruta y el verbo.
Arreglo: `WithName(...)` en Minimal APIs; en controladores, el nombre de la acción (o `Name` en el atributo de verbo).

-----

## Según la versión de .NET

* **ASP.NET Core 2.x–8:** las plantillas incluyen Swashbuckle (`AddSwaggerGen`, `UseSwaggerUI`).
* **ASP.NET Core 7:** `WithOpenApi()` y metadatos de `TypedResults` para Minimal APIs.
* **ASP.NET Core 9:** `Microsoft.AspNetCore.OpenApi` con `AddOpenApi`/`MapOpenApi`; las plantillas quitan Swashbuckle y no incluyen interfaz gráfica.
* **ASP.NET Core 10:** documentos OpenAPI 3.1 por defecto, salida en YAML y lectura de comentarios XML en el generador integrado.

-----

## Cuándo sí y cuándo no

**Usa el generador integrado cuando:** empiezas un proyecto en .NET 9 o posterior. Es el camino que mantiene Microsoft.

**Usa Swashbuckle o NSwag cuando:** el proyecto ya los usa y depende de sus extensiones, o necesitas algo que el integrado todavía no hace. Uno solo por proyecto.

**Expón el documento en producción cuando:** la API es pública o la consumen otros equipos. **No lo hagas cuando:** es interna y nadie lo necesita, sabiendo que eso no reemplaza la autenticación.

-----

## Resumen en 5 líneas

1. OpenAPI es el documento estándar; Swagger UI y Scalar solo lo muestran.
2. .NET 9+: `AddOpenApi()` + `MapOpenApi()`; la interfaz gráfica se agrega aparte.
3. La documentación es tan buena como tus tipos: `ActionResult<T>`, `TypedResults`, `Results<...>`.
4. `WithName`, `WithSummary` y `WithTags` mejoran el documento y los clientes generados.
5. Ocultar el documento no protege la API; protegerla es trabajo de la autenticación.

-----

## Para profundizar

<details>
<summary>Varios documentos y versiones</summary>

`builder.Services.AddOpenApi("v1")` y `AddOpenApi("v2")` generan documentos separados (`/openapi/v1.json`, `/openapi/v2.json`). Con `Asp.Versioning` y su integración de OpenAPI, cada versión de la API puede tener su documento ([Versionamiento de APIs](../01-diseno-de-apis-rest/03-Versionamiento%20de%20APIs.md)). También puedes filtrar qué endpoints entran en cada documento.

</details>

<details>
<summary>Transformadores: modificar el documento generado</summary>

`AddOpenApi(o => o.AddDocumentTransformer(...))` permite cambiar el documento completo (por ejemplo, agregar el esquema de seguridad Bearer y servidores), y `AddOperationTransformer` o `AddSchemaTransformer` cambian operaciones o esquemas individuales. Es el equivalente de los filtros de Swashbuckle. La API de los modelos cambió entre .NET 9 y .NET 10 (actualización de la biblioteca Microsoft.OpenApi), así que verifica ejemplos según tu versión.

</details>

-----

## En entrevista

### Respuesta corta (junior)

OpenAPI es el estándar que describe una API: rutas, parámetros y respuestas. Swagger UI es una página que lo muestra y permite probar la API. En .NET 9 o posterior se genera con `AddOpenApi()` y `MapOpenApi()`.

### Respuesta ampliada (semi-senior)

Distingo el documento OpenAPI de la interfaz que lo muestra. En .NET 9+ uso el generador integrado y agrego Scalar o Swagger UI si hace falta. Cuido la calidad del documento con tipos: `ActionResult<T>` o `ProducesResponseType` en controladores, `TypedResults` y `Results<...>` en Minimal APIs, y `WithName` para `operationId` estables. Lo genero en el build para revisar cambios de contrato y generar clientes en CI. Sé que ocultarlo en producción no es seguridad: la protección está en la autorización de cada endpoint.

### Preguntas frecuentes de seguimiento

**1. ¿Swagger y OpenAPI son lo mismo?**
No: OpenAPI es la especificación; Swagger es una familia de herramientas que la usa.

**2. ¿Por qué mi endpoint aparece sin esquema de respuesta?**
Porque devuelve `IActionResult` o `IResult` sin metadatos de tipo.

**3. ¿Qué cambió en .NET 9?**
El generador integrado reemplazó a Swashbuckle en las plantillas, sin interfaz gráfica incluida.

**4. ¿Para qué sirve `operationId`?**
Para nombrar la operación de forma estable; los generadores de clientes lo usan como nombre del método.

-----

## Práctica

**Ejercicio 1.** Este endpoint aparece en la documentación con `200` sin esquema. Corrígelo para que documente `200` con `TareaDto`, `404` y `400`, sin atributos.

```csharp
app.MapPut("/tareas/{id:int}", (int id, ActualizarTareaRequest r, ITareaService s) =>
{
    if (string.IsNullOrWhiteSpace(r.Titulo)) return Results.BadRequest();
    var tarea = s.Actualizar(id, r.Titulo);
    return tarea is null ? Results.NotFound() : Results.Ok(tarea);
});
```

<details>
<summary>Solución</summary>

```csharp
app.MapPut("/tareas/{id:int}",
    Results<Ok<TareaDto>, NotFound, ValidationProblem> (int id, ActualizarTareaRequest r, ITareaService s) =>
    {
        if (string.IsNullOrWhiteSpace(r.Titulo))
        {
            return TypedResults.ValidationProblem(new Dictionary<string, string[]>
            {
                [nameof(r.Titulo)] = ["El título es obligatorio."]
            });
        }

        TareaDto? tarea = s.Actualizar(id, r.Titulo);
        return tarea is null ? TypedResults.NotFound() : TypedResults.Ok(tarea);
    })
   .WithName("ActualizarTarea");
```

El tipo `Results<Ok<TareaDto>, NotFound, ValidationProblem>` le dice al generador los tres resultados posibles y el esquema del `200`. Además, `ValidationProblem` responde con el formato estándar en lugar de un `400` vacío.

</details>

**Ejercicio 2.** Agrega Scalar al ejemplo y haz que `dotnet run` abra el navegador directamente en la documentación.

<details>
<summary>Solución</summary>

```text
dotnet add package Scalar.AspNetCore
```

```csharp
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();   // GET /scalar
}
```

En `Properties/launchSettings.json`, en el perfil que uses:

```json
"launchBrowser": true,
"launchUrl": "scalar"
```

Scalar lee `/openapi/v1.json` por defecto. Como ambos se mapean solo en Development, en producción no existe ni el documento ni la interfaz.

</details>

-----

## Siguiente lección

Terminaste el módulo. Vuelve al [índice](README.md) o continúa con [Emisión de JWT y hash de contraseñas](../03-seguridad-de-apis/04-Emision%20de%20JWT%20y%20hash%20de%20contrasenas.md).
