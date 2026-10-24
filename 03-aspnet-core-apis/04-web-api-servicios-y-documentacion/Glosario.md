# Glosario: Web API, servicios y documentación

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**ActionResult&lt;T&gt;:** tipo de retorno que permite devolver un `T` (200) o cualquier otro resultado HTTP, y que documenta el tipo del cuerpo. ([Controladores](01-Controladores%20y%20rutas%20por%20atributo.md))

**Categoría (logging):** nombre que identifica quién escribe un log; con `ILogger<T>`, es el nombre completo de `T`. ([Logging](04-Logging%20con%20ILogger.md))

**Ciclo de vida (lifetime):** cuánto vive una instancia en el contenedor: toda la aplicación (singleton), una petición (scoped) o una resolución (transient). ([DI](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md))

**Contenedor de DI:** componente que sabe construir cada servicio registrado y sus dependencias. ([DI](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md))

**Cortocircuito (short-circuit):** responder desde un middleware sin llamar a `next`, de modo que el resto del pipeline no se ejecuta. ([Middleware](03-Middleware%20personalizado.md))

**Dependencia cautiva (captive dependency):** servicio de vida corta atrapado dentro de uno de vida más larga, como un scoped dentro de un singleton. ([DI](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md))

**Documento OpenAPI:** archivo JSON o YAML con la descripción completa de una API. ([OpenAPI](05-Documentacion%20con%20OpenAPI.md))

**Esquema (schema):** descripción de la forma de un JSON dentro de un documento OpenAPI. ([OpenAPI](05-Documentacion%20con%20OpenAPI.md))

**Inyección de dependencias (DI):** técnica en la que una clase recibe sus dependencias desde afuera, normalmente por constructor. ([DI](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md))

**Log estructurado:** registro que guarda cada valor como campo consultable, no solo como texto. ([Logging](04-Logging%20con%20ILogger.md))

**Middleware:** componente del pipeline HTTP que procesa la petición y decide si pasarla al siguiente. ([Middleware](03-Middleware%20personalizado.md))

**Nivel de log:** severidad de un mensaje, de `Trace` (0) a `Critical` (5). ([Logging](04-Logging%20con%20ILogger.md))

**OpenAPI:** especificación estándar para describir APIs HTTP; antes se llamaba Swagger Specification. ([OpenAPI](05-Documentacion%20con%20OpenAPI.md))

**Plantilla de mensaje (message template):** texto de log con marcadores con nombre (`{PedidoId}`) cuyos valores se guardan como propiedades. ([Logging](04-Logging%20con%20ILogger.md))

**Plantilla de ruta:** texto como `api/tareas/{id:int}` que define qué URLs coinciden con un endpoint. ([Controladores](01-Controladores%20y%20rutas%20por%20atributo.md))

**Provider (logging):** componente que envía los logs a un destino: consola, archivo o plataforma de logs. ([Logging](04-Logging%20con%20ILogger.md))

**RequestDelegate:** delegado que representa el resto del pipeline; llamarlo es `next(context)`. ([Middleware](03-Middleware%20personalizado.md))

**Respuesta iniciada:** momento en que los encabezados ya se enviaron (`Response.HasStarted`); desde ahí no se pueden cambiar. ([Middleware](03-Middleware%20personalizado.md))

**Restricción de ruta:** condición sobre un parámetro, como `{id:int}`, que decide si la URL coincide. ([Controladores](01-Controladores%20y%20rutas%20por%20atributo.md))

**Scope:** unidad de vida de los servicios scoped; en ASP.NET Core, una petición HTTP. ([DI](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md))

**Service locator:** antipatrón en el que una clase pide servicios al contenedor en lugar de recibirlos por constructor. ([DI](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md))

**Sink:** en Serilog, destino de los logs (archivo, Seq, Elasticsearch, base de datos). ([Logging](04-Logging%20con%20ILogger.md))

**Swagger:** familia de herramientas (Swagger UI, Swagger Editor) que trabajan con documentos OpenAPI. ([OpenAPI](05-Documentacion%20con%20OpenAPI.md))

**Swagger UI / Scalar:** interfaces web que leen un documento OpenAPI para explorar y probar la API. ([OpenAPI](05-Documentacion%20con%20OpenAPI.md))

**Token de reemplazo:** marcador como `[controller]` o `[action]` que ASP.NET Core sustituye por el nombre del controlador o del método. ([Controladores](01-Controladores%20y%20rutas%20por%20atributo.md))
