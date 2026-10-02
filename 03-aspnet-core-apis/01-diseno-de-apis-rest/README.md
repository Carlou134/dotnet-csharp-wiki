# Diseño de APIs REST

En esta carpeta aprendes a diseñar una API como un **contrato** estable y predecible: recursos y métodos HTTP bien usados, códigos de estado correctos, errores en formato estándar (**ProblemDetails**), **versionamiento** para evolucionar sin romper clientes y **HATEOAS** para que la API guíe al cliente por las acciones disponibles.

-----

## Antes de empezar

Conviene que ya tengas:

* C# moderno: records, `async`/`await`, excepciones: [C# Core](../../01-csharp-core-and-runtime/README.md).
* Modelo de dominio con reglas y estados: [DDD táctico](../../02-architecture-and-design-patterns/01-domain-driven-design/README.md).
* El SDK de .NET 10 y un editor que ejecute archivos `.http` (Visual Studio o VS Code con REST Client / C# Dev Kit).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Principios REST y códigos de estado](01-Principios%20REST%20y%20codigos%20de%20estado.md) | Recursos, métodos HTTP, idempotencia, códigos de estado, `[ApiController]`, `ActionResult<T>`, `CreatedAtAction`, DTOs | C# Core |
| [2. Errores con ProblemDetails](02-Errores%20con%20ProblemDetails.md) | RFC 9457, `Problem()`, `ValidationProblem`, `AddProblemDetails`, `IExceptionHandler`, el debate del envoltorio `{ success, data }` | Principios REST |
| [3. Versionamiento de APIs](03-Versionamiento%20de%20APIs.md) | Cambios que rompen, estrategias (URL, query, header), `Asp.Versioning`, `[MapToApiVersion]`, deprecación | Principios REST, ProblemDetails |
| [4. HATEOAS](04-HATEOAS.md) | Enlaces según el estado, `Url.Action`/`LinkGenerator`, formatos (HAL, Siren), paginación | Versionamiento, estados de un agregado |

Para practicar más: [Ejercicios de diseño de APIs](Ejercicios.md), con los 8 ejercicios de la Sesión 9 corregidos y 3 retos (todos con la solución plegada).

-----

## El mapa completo en una mirada

```
Recursos            /api/pedidos · /api/pedidos/5 · /api/pedidos/5/lineas   (plural, sin verbos)
Métodos             GET leer (seguro) · POST crear · PUT reemplazar · PATCH modificar · DELETE borrar
Idempotentes        GET · PUT · DELETE        (POST no → idempotency keys)
Códigos             200 · 201 + Location · 204 · 400 · 401 · 403 · 404 · 409 · 422 · 500

Éxito               2xx + el recurso
Error               4xx/5xx + ProblemDetails { type, title, status, detail, instance, traceId }
Problem(...)        ¡argumentos con nombre! el primero posicional es detail
Global              AddProblemDetails · UseExceptionHandler · UseStatusCodePages · IExceptionHandler

Versionar           solo cambios que rompen · Asp.Versioning.Mvc · AddApiVersioning().AddMvc()
                    controlador: [ApiVersion("1.0")] [ApiVersion("2.0")] · acción: [MapToApiVersion("2.0")]

HATEOAS             links según el ESTADO · Url.Action / LinkGenerator (nunca a mano) · rel + href + method
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura que en [C# Core](../../01-csharp-core-and-runtime/README.md): en una frase, antes de empezar, el problema, cómo funciona, ejemplo completo, errores comunes, según la versión de .NET, cuándo sí y cuándo no, resumen en 5 líneas, para profundizar, en entrevista, práctica y siguiente lección.

Los ejemplos completos son el `Program.cs` de un proyecto `dotnet new web`: copias, ejecutas con `dotnet run` y pruebas con un archivo `.http`.

-----

## Después de esta carpeta

Siguiente módulo: [Minimal APIs](../02-minimal-apis/README.md), la forma liviana de implementar estos mismos endpoints sin controladores. Después, los siguientes pasos son conectar la API con datos reales (EF Core, repositorios, transacciones), autenticación y autorización, y documentación con OpenAPI.
