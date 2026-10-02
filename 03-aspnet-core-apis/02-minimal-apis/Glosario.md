# Glosario: minimal APIs

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**`[AsParameters]`:** atributo que enlaza cada propiedad de un tipo como si fuera un parámetro suelto del handler. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**Binding (enlace de parámetros):** proceso que llena los parámetros del handler desde la ruta, la query string, los headers, el cuerpo o los servicios. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**Cortocircuito (*short-circuit*):** un filtro devuelve una respuesta sin llamar a `next`; el handler no se ejecuta. ([Endpoint filters](03-Endpoint%20filters.md))

**Endpoint filter:** componente que envuelve la ejecución de un handler, con acceso a sus argumentos y a su resultado. ([Endpoint filters](03-Endpoint%20filters.md))

**`EndpointFilterInvocationContext`:** contexto de un filtro: `HttpContext`, `Arguments` y `GetArgument<T>(índice)`. ([Endpoint filters](03-Endpoint%20filters.md))

**Filter factory:** fábrica de filtros que se ejecuta una vez por endpoint al iniciar y puede inspeccionar la firma del handler. ([Endpoint filters](03-Endpoint%20filters.md))

**Grupo de rutas (`MapGroup`):** conjunto de endpoints con un prefijo común y configuración compartida (tags, filtros, autorización). ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**Handler (manejador):** la función que atiende la petición de un endpoint. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**`IEndpointFilter`:** interfaz para escribir un filtro como clase reutilizable. ([Endpoint filters](03-Endpoint%20filters.md))

**`IEndpointMetadataProvider`:** interfaz que implementan los Typed Results para informar a OpenAPI su código y tipo de respuesta. ([Typed Results](02-Typed%20Results.md))

**`IResult`:** interfaz de todo resultado HTTP en minimal APIs. ([Typed Results](02-Typed%20Results.md))

**Metadatos del endpoint:** información asociada a un endpoint (nombre, tags, autorización, respuestas) que usan el ruteo, la seguridad y OpenAPI. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**Middleware:** componente del pipeline que procesa todas las peticiones antes y después del ruteo. ([Endpoint filters](03-Endpoint%20filters.md))

**Minimal API:** forma de definir endpoints HTTP con métodos `Map*` y funciones, sin controladores. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**OpenAPI:** especificación que describe una API HTTP para documentación y generación de clientes. ([Typed Results](02-Typed%20Results.md))

**Request Delegate Generator (RDG):** generador de código que crea en compilación el código de cada endpoint; permite Native AOT. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**Restricción de ruta:** condición sobre un parámetro de ruta (`{id:int}`, `{id:guid}`, `{id:min(1)}`) que decide si la ruta coincide. ([Endpoints](01-Endpoints%20y%20grupos%20de%20rutas.md))

**`Results` (clase estática):** fábrica de resultados que devuelve `IResult`. ([Typed Results](02-Typed%20Results.md))

**`Results<T1, ..., T6>`:** unión que declara las respuestas posibles de un handler. ([Typed Results](02-Typed%20Results.md))

**`TypedResults` (clase estática):** fábrica de resultados que devuelve el tipo concreto (`Ok<T>`, `NotFound`...). ([Typed Results](02-Typed%20Results.md))

**Validación integrada (.NET 10):** `builder.Services.AddValidation()` valida los atributos de DataAnnotations de los parámetros de minimal APIs y responde 400 con ProblemDetails. ([Endpoint filters](03-Endpoint%20filters.md))
