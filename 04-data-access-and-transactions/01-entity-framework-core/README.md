# Entity Framework Core

Este módulo convierte los apuntes de CRUD y Code First en una base actual para .NET 10: contexto de vida corta, configuración segura, proveedores explícitos, migraciones revisables y consultas asíncronas.

-----

## Antes de empezar
Conviene conocer [ASP.NET Core](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/README.md), clases, DI y `async`/`await`.

-----

## Orden de lectura
| Lección | Qué aprendes |
| --- | --- |
| [1. DbContext, entidades y configuración](01-DbContext%20entidades%20y%20configuracion.md) | Proveedor, cadena de conexión, `DbSet`, DI y unidad de trabajo |
| [2. Migraciones y consultas](02-Migraciones%20y%20consultas.md) | Code First, herramientas, revisión del esquema y lectura asíncrona |
| [3. CRUD seguro con MVC y EF Core](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md) | ViewModels, validación, antiforgery, overposting y Post/Redirect/Get |

-----

## El mapa completo en una mirada
```text
entidades + configuración -> modelo EF Core
modelo anterior + modelo actual -> migración revisable
migración -> esquema de base de datos

request -> DbContext scoped -> consulta/cambio -> SaveChangesAsync -> dispose
```

-----

## Cómo está armada cada lección
Cada lección usa el formato de 13 secciones. Los ejemplos usan SQLite para ser multiplataforma; SQL Server requiere su proveedor y una conexión apropiada.

-----

## Después de esta carpeta
Continúa con relaciones, consultas eficientes, transacciones, concurrencia optimista e índices. No confundas completar un CRUD con diseñar correctamente la persistencia.
