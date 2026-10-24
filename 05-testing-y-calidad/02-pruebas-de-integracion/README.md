# Pruebas de integración

En esta carpeta pasas de probar piezas a probar el sistema armado: la API completa en memoria con `WebApplicationFactory`, servicios externos, autenticación y reloj reemplazados de forma controlada, una base de datos real en Docker con Testcontainers y pruebas aisladas entre sí con Respawn.

-----

## Antes de empezar

Conviene que ya tengas:

* Pruebas unitarias con xUnit: [Pruebas unitarias](../01-pruebas-unitarias/README.md).
* Minimal APIs y seguridad: [Minimal APIs](../../03-aspnet-core-apis/02-minimal-apis/README.md) y [Seguridad de APIs](../../03-aspnet-core-apis/03-seguridad-de-apis/README.md).
* EF Core: [Entity Framework Core](../../04-data-access-and-transactions/01-entity-framework-core/README.md).
* **Docker** instalado y en ejecución para las lecciones 3 y 4.

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. WebApplicationFactory y pruebas de API en memoria](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md) | `TestServer`, `CreateClient`, qué verificar por HTTP, contratos y estado compartido | xUnit, Minimal APIs |
| [2. Reemplazar servicios y autenticación en pruebas](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md) | `ConfigureTestServices`, `WithWebHostBuilder`, handler de autenticación de prueba, `FakeTimeProvider` | Lección 1, autenticación y DI |
| [3. Base de datos real con Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md) | Por qué no InMemory, PostgreSQL en Docker, `IAsyncLifetime`, cadena por configuración | Lección 1, EF Core, Docker |
| [4. Datos de prueba y aislamiento con Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md) | Estrategias de aislamiento, Respawn, colecciones, datos sembrados | Lección 3 |

Para practicar: [Ejercicios de pruebas de integración](Ejercicios.md), con tres ejercicios guiados, dos retos (migraciones y CI) y un checkpoint.

-----

## El mapa completo en una mirada

```text
Proceso de pruebas
  prueba ──HttpClient──► TestServer ──► Program.cs REAL (config, DI, middleware, endpoints)
                                              │
              ConfigureTestServices ──────────┤  reemplaza: auth (esquema "Test"), reloj, servicios externos
              UseSetting(cadena) ─────────────┤
                                              ▼
                                   PostgreSQL en Docker (Testcontainers, puerto aleatorio)

Ciclo por colección:  StartAsync (contenedor) → construir app → EnsureCreated / Migrate → Respawner.CreateAsync
Ciclo por prueba:     ResetAsync (base vacía) → preparar datos → llamar por HTTP → verificar respuesta y base

InMemory ✗ (sin índices, sin SQL, sin transacciones) · SQLite ~ · Testcontainers ✔ mismo motor que producción
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki y usan la misma API de tareas, que crece de una lección a la siguiente (en memoria → autenticación → PostgreSQL → aislamiento). Los resúmenes de `dotnet test` se describen por cantidad de pruebas porque su formato depende de la versión del SDK y del idioma.

-----

## Después de esta carpeta

Con pruebas de integración en marcha, el siguiente paso del roadmap es organizar la solución con [Clean y Hexagonal Architecture](../../02-architecture-and-design-patterns/07-clean-y-hexagonal/README.md): capas bien separadas hacen que estas pruebas sean todavía más simples. También puedes sumar pruebas de contrato (Pact) cuando la API tiene consumidores de otros equipos.
