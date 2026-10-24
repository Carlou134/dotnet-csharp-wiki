# Web API: servicios, middleware y documentación

En esta carpeta armas las piezas que sostienen cualquier Web API de ASP.NET Core: controladores con rutas estables, inyección de dependencias con el ciclo de vida correcto, middleware propio que no rompe respuestas, logs que sirven para diagnosticar y un documento OpenAPI que describe la API de verdad.

-----

## Antes de empezar

Conviene que ya tengas:

* Host, configuración y pipeline: [Fundamentos de ASP.NET Core](../00-fundamentos-de-aspnet-core/README.md).
* Recursos, verbos y códigos de estado: [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md).
* `TypedResults` y endpoint filters: [Minimal APIs](../02-minimal-apis/README.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Controladores y rutas por atributo](01-Controladores%20y%20rutas%20por%20atributo.md) | `[ApiController]`, plantillas y tokens de ruta, restricciones, `ActionResult<T>` y controladores delgados | REST, modelos de aplicación |
| [2. Inyección de dependencias y ciclos de vida](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md) | Singleton, scoped y transient según el estado; dependencias cautivas; scopes manuales; keyed services | Constructores, interfaces |
| [3. Middleware personalizado](03-Middleware%20personalizado.md) | Antes, después y en lugar de `next`; `OnStarting`; convención frente a `IMiddleware` | Pipeline, DI |
| [4. Logging con ILogger](04-Logging%20con%20ILogger.md) | Niveles, categorías, plantillas, `[LoggerMessage]`, providers y qué no registrar | DI, configuración |
| [5. Documentación con OpenAPI](05-Documentacion%20con%20OpenAPI.md) | `AddOpenApi`/`MapOpenApi`, Scalar y Swagger UI, metadatos desde los tipos | Controladores, Typed Results |

-----

## El mapa completo en una mirada

```text
petición ─► middleware (antes) ─► routing ─► auth ─► controlador / endpoint ─► servicio (DI)
                                                         │
respuesta ◄─ middleware (después: solo leer) ◄──────────┘

DI: singleton (estado de la app, seguro entre hilos) · scoped (estado de la petición) · transient (sin estado)
    nunca un lifetime más largo que sus dependencias

ILogger<T>: plantilla + nivel ─► filtros por categoría ─► provider (consola, Serilog, OTLP)

Tipos (ActionResult<T>, Results<...>) ─► documento OpenAPI ─► Scalar / Swagger UI / clientes generados
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki con .NET 10. Las respuestas HTTP abrevian valores que cambian en cada ejecución (puertos, tiempos, fechas).

-----

## Después de esta carpeta

Protege la API con [Seguridad de APIs](../03-seguridad-de-apis/README.md), incluida la [emisión de JWT propios](../03-seguridad-de-apis/04-Emision%20de%20JWT%20y%20hash%20de%20contrasenas.md). Para que lo que construiste no se rompa al cambiarlo, sigue con [Pruebas unitarias](../../05-testing-y-calidad/01-pruebas-unitarias/README.md).
