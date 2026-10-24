# Clean y Hexagonal Architecture

En esta carpeta unes las piezas que ya tienes (DDD, CQRS, EF Core, DI, pruebas) en una solución con una regla clara: las dependencias apuntan hacia el dominio. Aprendes la regla, el vocabulario de puertos y adaptadores, cómo se arma una solución real en .NET con cuatro proyectos y cómo proteger la arquitectura con pruebas, y también cuándo **no** usarla.

-----

## Antes de empezar

Conviene que ya tengas:

* Entidades, value objects y agregados: [Domain-Driven Design táctico](../01-domain-driven-design/README.md).
* Commands, queries y casos de uso: [CQRS y mediator](../06-cqrs-y-mediator/README.md).
* EF Core con Fluent API: [Entity Framework Core](../../04-data-access-and-transactions/01-entity-framework-core/README.md).
* Inyección de dependencias: [Web API: servicios, middleware y documentación](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/README.md).
* Pruebas unitarias y de integración: [Testing y calidad](../../05-testing-y-calidad/README.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Capas y la regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md) | N-capas frente a Clean, círculos, inversión de dependencias, raíz de composición y lo que Clean no es | DDD, SOLID |
| [2. Puertos y adaptadores](02-Puertos%20y%20adaptadores.md) | Puertos de entrada y salida, adaptadores, el hexágono, pruebas como adaptadores | Lección 1 |
| [3. Estructura de una solución Clean en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md) | Cuatro proyectos y sus referencias, `AddX()` por capa, `internal`, mapeo con Fluent API, DTOs y errores a HTTP | Lecciones 1 y 2, EF Core |
| [4. Pruebas de arquitectura y cuándo no usar Clean](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md) | NetArchTest, erosión, Vertical Slice, monolito modular y señales de sobrediseño | Lección 3, xUnit |

Para practicar: [Ejercicios de Clean y Hexagonal](Ejercicios.md), con tres ejercicios guiados, dos retos y un checkpoint.

-----

## El mapa completo en una mirada

```text
                    Tienda.Api  (raíz de composición + adaptadores de entrada HTTP)
                     │        │
                     ▼        ▼
   Tienda.Aplicacion ◄──── Tienda.Infraestructura  (adaptadores de salida: EF Core, HTTP, SMTP)
   casos de uso + puertos       implementa los puertos · tipos internal
                     │
                     ▼
              Tienda.Dominio  (entidades ricas, reglas · sin paquetes)

Regla: las dependencias del código apuntan hacia adentro.
Puertos de entrada: casos de uso · Puertos de salida: repositorio, pasarela, email
Adaptadores: endpoints, jobs, pruebas (entrada) · EF Core, clientes HTTP (salida)
Protección: referencias de proyecto + internal + pruebas de arquitectura (NetArchTest)
Criterio: reglas de negocio reales → Clean · CRUD → capas simples o Vertical Slice
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki y usan el agregado `Pedido` del módulo de DDD. Las lecciones 1 y 2 muestran las capas en una consola de un solo archivo; la 3 arma la solución completa con cuatro proyectos, y la 4 la protege con pruebas.

-----

## Después de esta carpeta

El siguiente paso del roadmap es que esta arquitectura aguante carga: concurrencia optimista, índices y planes de ejecución, y caché (planificado en el bloque de acceso a datos). Más adelante, cuando un sistema crece en áreas separables, el monolito modular es el paso previo a considerar microservicios.
