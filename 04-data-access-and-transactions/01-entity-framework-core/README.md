# Entity Framework Core

Este módulo convierte los apuntes de ADO.NET, ORM, CRUD y Code First en una base actual para .NET 10: qué hace un ORM por debajo, contexto de vida corta, modelado explícito, migraciones revisables, relaciones cargadas a propósito y CRUD que no confía en lo que manda el cliente.

-----

## Antes de empezar

Conviene conocer [ASP.NET Core](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/README.md), clases, DI, LINQ y `async`/`await`.

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes |
| --- | --- |
| [1. ADO.NET, ORMs y EF Core](01-ADO%20NET%20ORMs%20y%20EF%20Core.md) | Arquitectura cliente-backend-base, proveedores ADO.NET, inyección SQL, ORM frente a micro-ORM, EF6 frente a EF Core |
| [2. DbContext, entidades y configuración](02-DbContext%20entidades%20y%20configuracion.md) | Proveedor, cadena de conexión, `DbSet`, DI y unidad de trabajo |
| [3. Modelado con convenciones, atributos y Fluent API](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md) | Precedencia de configuración, `IEntityTypeConfiguration`, esquemas, claves `Guid`, enums y `HasData` |
| [4. Migraciones y consultas](04-Migraciones%20y%20consultas.md) | Code First, herramientas, revisión del esquema y lectura asíncrona |
| [5. Relaciones y carga de datos](05-Relaciones%20y%20carga%20de%20datos.md) | 1:1, 1:N, N:N, `Include`, lazy loading y N+1, `AsSplitQuery` y proyecciones |
| [6. CRUD en APIs con EF Core](06-CRUD%20en%20APIs%20con%20EF%20Core.md) | DTOs de entrada, `Find`/`Single`, `201 Created`, `ExecuteUpdateAsync` y `ExecuteDeleteAsync` |
| [7. CRUD seguro con MVC y EF Core](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md) | ViewModels, validación, antiforgery, overposting y Post/Redirect/Get |

-----

## El mapa completo en una mirada

```text
cliente ──HTTP──► backend ──EF Core──► ADO.NET ──► base de datos
                              │
convenciones < atributos < Fluent API  ──►  modelo EF Core
modelo anterior + modelo actual        ──►  migración revisable ──► esquema

request ─► DbContext scoped ─► consulta (proyección, Include, AsSplitQuery)
                           └─► cambio (Add / modificar rastreada / ExecuteUpdate) ─► SaveChangesAsync ─► dispose

Navegaciones: no se cargan solas · lazy loading = N+1 · varias colecciones = AsSplitQuery
```

-----

## Cómo está armada cada lección

Cada lección usa el formato de 13 secciones. Los ejemplos usan SQLite para ser multiplataforma; SQL Server requiere su proveedor y una conexión apropiada. Las salidas de consola son las que produce el código; las respuestas HTTP abrevian los Guid y el puerto porque cambian en cada ejecución.

-----

## Después de esta carpeta

Continúa con [ADO.NET y Dapper](../02-ado-net-y-dapper/README.md): lo que hay debajo de EF Core, SQL propio con Dapper y transacciones que comparten conexión. Si tus casos de uso crecen, mira cómo organizarlos con DTOs, commands, queries y validación en [CQRS y mediator](../../02-architecture-and-design-patterns/06-cqrs-y-mediator/README.md). No confundas completar un CRUD con diseñar correctamente la persistencia.
