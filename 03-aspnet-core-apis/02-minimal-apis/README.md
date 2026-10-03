# Minimal APIs

En esta carpeta aprendes a construir APIs con **minimal APIs** de .NET 10: endpoints definidos con `MapGet`/`MapPost`, organizados en **grupos de rutas**; respuestas declaradas en la firma con **Typed Results**, y lógica transversal (validación, logging, reglas por recurso) con **endpoint filters**.

-----

## Antes de empezar

Conviene que ya tengas:

* El diseño de APIs REST (recursos, códigos de estado, ProblemDetails): [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md).
* Lambdas, delegados y métodos de extensión: [C# Core](../../01-csharp-core-and-runtime/README.md).
* El SDK de .NET 10.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Endpoints y grupos de rutas](01-Endpoints%20y%20grupos%20de%20rutas.md) | `MapGet`/`MapPost`, binding de parámetros, restricciones de ruta, `MapGroup`, organizar por recurso | Diseño de APIs REST |
| [2. Typed Results](02-Typed%20Results.md) | `TypedResults` frente a `Results`, uniones `Results<...>`, OpenAPI automático, pruebas de handlers | Endpoints y grupos |
| [3. Endpoint filters](03-Endpoint%20filters.md) | `AddEndpointFilter`, `IEndpointFilter`, orden, cortocircuito, filtro frente a middleware, validación integrada de .NET 10 | Typed Results, ProblemDetails |

Para practicar más: [Ejercicios de minimal APIs](Ejercicios.md), con los 5 ejercicios y 3 retos de la Sesión 10 corregidos (todos con la solución plegada).

-----

## El mapa completo en una mirada

```
Endpoint            app.MapGet("/ruta/{id:int}", handler)        lo que devuelve el handler = respuesta
Binding             ruta · query string · cuerpo (1 tipo complejo) · servicios DI · HttpContext · CancellationToken
Grupo               app.MapGroup("/api/x").WithTags(..).RequireAuthorization().AddEndpointFilter<F>()
Organizar           un método de extensión MapX por recurso · handlers como métodos con nombre

TypedResults        TypedResults.Ok(x) → Ok<T>     (Results.Ok(x) → IResult: mismo objeto, menos información)
Unión               Results<Ok<T>, NotFound, ValidationProblem>   (2 a 6 tipos; devolver otro → CS0029)
Ojo con nombres     UnauthorizedHttpResult · ForbidHttpResult · ProblemHttpResult
Beneficios          contrato verificado · OpenAPI automático · pruebas sin servidor   (NO rendimiento)

Filtro              antes → (cortocircuito | await next) → después     acceso a Arguments y al resultado
Orden               grupo envuelve a endpoint · primero registrado = más externo
No para             autenticación (RequireAuthorization) · excepciones · cache · rate limiting
.NET 10             builder.Services.AddValidation() valida DataAnnotations sin filtro propio
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura que en [C# Core](../../01-csharp-core-and-runtime/README.md): en una frase, antes de empezar, el problema, cómo funciona, ejemplo completo, errores comunes, según la versión de .NET, cuándo sí y cuándo no, resumen en 5 líneas, para profundizar, en entrevista, práctica y siguiente lección.

Los ejemplos completos son el `Program.cs` de un proyecto `dotnet new web`.

-----

## Después de esta carpeta

Continúa con [Seguridad de APIs](../03-seguridad-de-apis/README.md) para agregar autenticación JWT bearer y autorización con políticas. Después puedes estudiar validación avanzada y conectar los endpoints a datos reales con EF Core.
