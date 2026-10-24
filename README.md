# ⚡ .NET & C# Enterprise Architecture Wiki

Repositorio personal de apuntes para el desarrollo backend con C# y .NET: desde los fundamentos del lenguaje hasta el diseño del dominio y de APIs. Cada lección sigue el mismo formato de estudio, con ejemplos que compilan, errores comunes, preguntas de entrevista y práctica con la solución plegada.

Versión de referencia: **.NET 10 / C# 14**.

-----

## 📂 Estructura

### 1. C# Core & Runtime ([`/01-csharp-core-and-runtime`](01-csharp-core-and-runtime/README.md))

12 módulos, 56 lecciones:

| Módulo | Temas |
| --- | --- |
| [00. Introducción](01-csharp-core-and-runtime/00-introduccion/README.md) | Plataforma .NET, CLR, compilación, sistema de tipos, SDK, CLI, proyectos y NuGet |
| [01. Tipos y variables](01-csharp-core-and-runtime/01-tipos-y-variables/README.md) | Tipos de valor y referencia, numéricos, texto, conversiones |
| [02. Control de flujo](01-csharp-core-and-runtime/02-control-de-flujo/README.md) | Condicionales, `switch` y patrones, bucles |
| [03. Métodos](01-csharp-core-and-runtime/03-metodos/README.md) | Parámetros, sobrecarga, `ref`/`out`, funciones locales |
| [04. POO](01-csharp-core-and-runtime/04-poo/README.md) | Clases, encapsulación, herencia, polimorfismo, interfaces |
| [05. Tipos avanzados](01-csharp-core-and-runtime/05-tipos-avanzados/README.md) | Enums, structs y records, nulos, genéricos, varianza |
| [06. Colecciones](01-csharp-core-and-runtime/06-colecciones/README.md) | Listas, diccionarios, conjuntos, colas, pilas, iteradores |
| [07. LINQ](01-csharp-core-and-runtime/07-linq/README.md) | Filtrar, ordenar, paginar, agrupar y unir |
| [08. Excepciones](01-csharp-core-and-runtime/08-excepciones/README.md) | Manejo, lanzamiento y excepciones propias |
| [09. Delegados y eventos](01-csharp-core-and-runtime/09-delegados-y-eventos/README.md) | Delegados, lambdas, eventos |
| [10. Asincronía y archivos](01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md) | `async`/`await`, cancelación, `IProgress<T>`, streams y archivos · [ejercicios](01-csharp-core-and-runtime/10-asincronia-y-archivos/Ejercicios.md) |
| [11. Código limpio](01-csharp-core-and-runtime/11-codigo-limpio/README.md) | Nombres, code smells, DRY/KISS/SOLID, sintaxis moderna |

### 2. Arquitectura & Patrones de Diseño ([`/02-architecture-and-design-patterns`](02-architecture-and-design-patterns/))

| Módulo | Temas |
| --- | --- |
| [01. Domain-Driven Design táctico](02-architecture-and-design-patterns/01-domain-driven-design/README.md) | Modelo anémico vs rico, entidades, value objects, agregados · [ejercicios](02-architecture-and-design-patterns/01-domain-driven-design/Ejercicios.md) |
| [02. Arquitectura dirigida por eventos](02-architecture-and-design-patterns/02-event-driven-architecture/README.md) | Eventos vs comandos, Pub/Sub en memoria, outbox, idempotencia, reintentos y DLQ · [ejercicios](02-architecture-and-design-patterns/02-event-driven-architecture/Ejercicios.md) |
| [03. Resiliencia de servicios con Polly](02-architecture-and-design-patterns/03-resiliencia-de-servicios/README.md) | Retry, backoff, jitter, circuit breaker, timeouts y pipelines HTTP con Polly 8 · [ejercicios](02-architecture-and-design-patterns/03-resiliencia-de-servicios/Ejercicios.md) |
| [04. Observabilidad con OpenTelemetry](02-architecture-and-design-patterns/04-observabilidad-con-opentelemetry/README.md) | Trazas distribuidas, métricas, logs estructurados, correlación y exportación OTLP · [ejercicios](02-architecture-and-design-patterns/04-observabilidad-con-opentelemetry/Ejercicios.md) |
| [05. Cloud native y contenedores](02-architecture-and-design-patterns/05-cloud-native-y-contenedores/README.md) | Docker, builds multi-stage, imágenes seguras, Native AOT y diseño cloud native · [ejercicios](02-architecture-and-design-patterns/05-cloud-native-y-contenedores/Ejercicios.md) |
| [06. CQRS y mediator](02-architecture-and-design-patterns/06-cqrs-y-mediator/README.md) | DTOs y mapeo, commands y queries con MediatR, notificaciones en proceso y validación con pipeline behaviors |
| [07. Clean y Hexagonal Architecture](02-architecture-and-design-patterns/07-clean-y-hexagonal/README.md) | Regla de dependencias, puertos y adaptadores, solución en cuatro proyectos, pruebas de arquitectura con NetArchTest y cuándo no usar Clean · [ejercicios](02-architecture-and-design-patterns/07-clean-y-hexagonal/Ejercicios.md) |

Planificado: mensajería con brokers, microservicios y patrones GoF (*Factory*, *Strategy*, *Repository*, *Decorator*, *Builder*).

### 3. ASP.NET Core ([`/03-aspnet-core-apis`](03-aspnet-core-apis/))

| Módulo | Temas |
| --- | --- |
| [00. Fundamentos de ASP.NET Core](03-aspnet-core-apis/00-fundamentos-de-aspnet-core/README.md) | Host, configuración, middleware, modelos de aplicación, MVC, routing y Razor |
| [01. Diseño de APIs REST](03-aspnet-core-apis/01-diseno-de-apis-rest/README.md) | Principios REST, códigos de estado, ProblemDetails, versionamiento, HATEOAS · [ejercicios](03-aspnet-core-apis/01-diseno-de-apis-rest/Ejercicios.md) |
| [02. Minimal APIs](03-aspnet-core-apis/02-minimal-apis/README.md) | Endpoints y grupos de rutas, Typed Results, endpoint filters · [ejercicios](03-aspnet-core-apis/02-minimal-apis/Ejercicios.md) |
| [03. Seguridad de APIs](03-aspnet-core-apis/03-seguridad-de-apis/README.md) | OAuth 2.0, OpenID Connect, JWT bearer, claims, roles y políticas, emisión de JWT propios y hash de contraseñas · [ejercicios](03-aspnet-core-apis/03-seguridad-de-apis/Ejercicios.md) |
| [04. Web API: servicios, middleware y documentación](03-aspnet-core-apis/04-web-api-servicios-y-documentacion/README.md) | Controladores y rutas por atributo, inyección de dependencias y ciclos de vida, middleware propio, logging con `ILogger`, OpenAPI |

Planificado: validación integrada de .NET 10, OWASP Top 10 y Health Checks.

### 4. Acceso a Datos & Transacciones ([`/04-data-access-and-transactions`](04-data-access-and-transactions/README.md))

| Módulo | Temas |
| --- | --- |
| [01. Entity Framework Core](04-data-access-and-transactions/01-entity-framework-core/README.md) | ADO.NET y ORMs, `DbContext`, modelado con Fluent API, migraciones, relaciones y carga de datos, CRUD en APIs y con MVC |
| [02. ADO.NET y Dapper](04-data-access-and-transactions/02-ado-net-y-dapper/README.md) | Pool de conexiones, readers, Dapper (multi-mapping, `QueryMultiple`), convivencia con EF Core y transacciones compartidas |
| [03. Rendimiento y concurrencia](04-data-access-and-transactions/03-rendimiento-y-concurrencia/README.md) | Concurrencia optimista a fondo, índices y planes de ejecución, consultas lentas y N+1 en producción, paginación por cursor · [ejercicios](04-data-access-and-transactions/03-rendimiento-y-concurrencia/Ejercicios.md) |

Planificado: caché (`IMemoryCache`, `HybridCache`, Redis).

### 5. Testing y Calidad ([`/05-testing-y-calidad`](05-testing-y-calidad/README.md))

| Módulo | Temas |
| --- | --- |
| [01. Pruebas unitarias](05-testing-y-calidad/01-pruebas-unitarias/README.md) | Pruebas unitarias y TDD, xUnit (`Fact`, `Theory`, aserciones), dobles de prueba con Moq, cobertura con Coverlet |
| [02. Pruebas de integración](05-testing-y-calidad/02-pruebas-de-integracion/README.md) | `WebApplicationFactory`, `ConfigureTestServices`, autenticación de prueba, `FakeTimeProvider`, Testcontainers y Respawn · [ejercicios](05-testing-y-calidad/02-pruebas-de-integracion/Ejercicios.md) |

Planificado: pruebas de contrato y end-to-end.

### 6. Legacy y Migración ([`/06-legacy-y-migracion`](06-legacy-y-migracion/README.md))

| Módulo | Temas |
| --- | --- |
| [01. Tecnologías heredadas](06-legacy-y-migracion/01-tecnologias-heredadas/README.md) | .NET Framework frente a .NET moderno, ASP.NET Web Forms, Windows Forms y DevExpress, WCF y CoreWCF, VB.NET para desarrolladores de C# |
| [02. IIS y migración](06-legacy-y-migracion/02-iis-y-migracion/README.md) | Despliegue en IIS (in-process, out-of-process, errores 500.3x) y estrategias de migración con strangler fig y YARP |

Planificado: WPF y MVVM, EF6 a EF Core, servicios de Windows frente a *worker services*.

-----

## 📖 Formato de las lecciones

Cada módulo tiene un `README.md` (orden de lectura y mapa del tema), un `Glosario.md` y, cuando hay material de clase, un `Ejercicios.md` con ejercicios guiados y retos.

Cada lección tiene las mismas 13 secciones:

1. **En una frase**
2. **Antes de empezar** (requisitos y palabras nuevas)
3. **El problema**
4. **Cómo funciona**
5. **Ejemplo completo** (código que compila)
6. **Errores comunes** (qué pasa, por qué y cómo se arregla)
7. **Según la versión** de C# o de .NET
8. **Cuándo sí y cuándo no**
9. **Resumen en 5 líneas**
10. **Para profundizar** (plegado)
11. **En entrevista** (respuesta junior, respuesta semi-senior y preguntas de seguimiento)
12. **Práctica** (solución plegada)
13. **Siguiente lección**
