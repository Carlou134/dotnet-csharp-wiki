# ⚡ .NET & C# Enterprise Architecture Wiki

Repositorio personal de apuntes para el desarrollo backend con C# y .NET: desde los fundamentos del lenguaje hasta el diseño del dominio y de APIs. Cada lección sigue el mismo formato de estudio, con ejemplos que compilan, errores comunes, preguntas de entrevista y práctica con la solución plegada.

Versión de referencia: **.NET 10 / C# 14**.

-----

## 📂 Estructura

### 1. C# Core & Runtime ([`/01-csharp-core-and-runtime`](01-csharp-core-and-runtime/README.md))

12 módulos, 53 lecciones:

| Módulo | Temas |
| --- | --- |
| [00. Introducción](01-csharp-core-and-runtime/00-introduccion/README.md) | Qué es C# y .NET, entorno, primer programa, paradigmas |
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

Planificado: patrones GoF (*Factory*, *Strategy*, *Repository*, *Decorator*, *Builder*), Clean y Hexagonal Architecture.

### 3. ASP.NET Core Web APIs ([`/03-aspnet-core-apis`](03-aspnet-core-apis/))

| Módulo | Temas |
| --- | --- |
| [01. Diseño de APIs REST](03-aspnet-core-apis/01-diseno-de-apis-rest/README.md) | Principios REST, códigos de estado, ProblemDetails, versionamiento, HATEOAS · [ejercicios](03-aspnet-core-apis/01-diseno-de-apis-rest/Ejercicios.md) |
| [02. Minimal APIs](03-aspnet-core-apis/02-minimal-apis/README.md) | Endpoints y grupos de rutas, Typed Results, endpoint filters · [ejercicios](03-aspnet-core-apis/02-minimal-apis/Ejercicios.md) |
| [03. Seguridad de APIs](03-aspnet-core-apis/03-seguridad-de-apis/README.md) | OAuth 2.0, OpenID Connect, JWT bearer, claims, roles y políticas · [ejercicios](03-aspnet-core-apis/03-seguridad-de-apis/Ejercicios.md) |

Planificado: validación (FluentValidation y validación integrada de .NET 10), inyección de dependencias y ciclos de vida, OWASP Top 10, logging estructurado con Serilog, Health Checks.

### 4. Acceso a Datos & Transacciones (`/04-data-access-and-transactions`)

Planificado: Entity Framework Core (mapeo, relaciones, *migrations*, `AsNoTracking`), transacciones ACID (`IDbContextTransaction`, `TransactionScope`), concurrencia optimista (`RowVersion`) y pesimista, rendimiento SQL (N+1, índices, planes de ejecución).

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
