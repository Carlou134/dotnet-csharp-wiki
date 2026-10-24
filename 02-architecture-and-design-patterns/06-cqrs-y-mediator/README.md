# CQRS y mediator

En esta carpeta aprendes a organizar los casos de uso de una API: contratos con DTOs que no exponen entidades, commands y queries con un handler cada uno, notificaciones para efectos secundarios y validación centralizada en un pipeline. MediatR es la herramienta de los ejemplos, pero el objetivo es entender los patrones, sus costos y cuándo **no** usarlos.

-----

## Antes de empezar

Conviene que ya tengas:

* EF Core, proyecciones y CRUD: [Entity Framework Core](../../04-data-access-and-transactions/01-entity-framework-core/README.md).
* Minimal APIs y DI: [Minimal APIs](../../03-aspnet-core-apis/02-minimal-apis/README.md).
* Eventos y bus en memoria: [Arquitectura dirigida por eventos](../02-event-driven-architecture/README.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. DTOs y mapeo](01-DTOs%20y%20mapeo.md) | DTOs de entrada y salida, proyección en la consulta, AutoMapper y sus alternativas | EF Core, proyecciones |
| [2. CQRS y mediator con MediatR](02-CQRS%20y%20mediator%20con%20MediatR.md) | Commands frente a queries, `IRequest`, handlers, *vertical slices* y costos del mediator | DTOs, DI |
| [3. Notificaciones en proceso con INotification](03-Notificaciones%20en%20proceso%20con%20INotification.md) | `Publish`, publicadores, errores en handlers y por qué en proceso no es confiable | CQRS, Pub/Sub en memoria |
| [4. Validación con pipeline behaviors](04-Validacion%20con%20pipeline%20behaviors.md) | FluentValidation, `IPipelineBehavior`, `IExceptionHandler` y niveles de validación | MediatR, ProblemDetails |

-----

## El mapa completo en una mirada

```text
HTTP ─► endpoint ─► DTO de entrada = Command ─► ISender.Send
                                                  │
                         LoggingBehavior ─► ValidationBehavior ─► Handler ─► entidad + SaveChanges
                                                  │ falla                      │
                                   ValidationException                        Publish(INotification)
                                                  │                            │
                         IExceptionHandler ─► 400 ProblemDetails     handlers de efectos secundarios
                                                                     (en proceso: no confiable → outbox)

HTTP ─► endpoint ─► Query ─► Handler ─► Select a DTO de salida (sin tracking)
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Los ejemplos usan MediatR 13, FluentValidation 12 y EF Core 10 con SQLite. MediatR y AutoMapper tienen licencia comercial desde 2025: cada lección indica alternativas.

-----

## Después de esta carpeta

Con los casos de uso separados, el paso natural es la arquitectura que los contiene: Clean y Hexagonal Architecture, con el dominio en el centro. Para efectos secundarios confiables, repasa [Entrega confiable e idempotencia](../02-event-driven-architecture/03-Entrega%20confiable%20e%20idempotencia.md).
