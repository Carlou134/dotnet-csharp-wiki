# Glosario: Entity Framework Core

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Antiforgery (CSRF):** protección que exige un token oculto en cada formulario para que otro sitio no pueda enviar peticiones en nombre del usuario. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Change tracker:** componente del `DbContext` que recuerda el estado (`Added`, `Unchanged`, `Modified`, `Deleted`) de cada entidad cargada. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Code First:** flujo donde el modelo de código y sus migraciones conducen la evolución del esquema. ([Migraciones](02-Migraciones%20y%20consultas.md))

**Concurrencia optimista:** estrategia que no bloquea al leer y detecta al guardar si otro proceso cambió la fila; EF Core lanza `DbUpdateConcurrencyException`. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**DbContext:** sesión de trabajo de EF Core que rastrea entidades y coordina consultas y cambios. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**DbSet:** puerta de entrada tipada para consultar y modificar entidades de un tipo. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Fluent API:** configuración del modelo con código en `OnModelCreating` o en clases `IEntityTypeConfiguration<T>`. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Migración:** conjunto versionado de operaciones (`Up` y `Down`) para llevar el esquema de un estado a otro. ([Migraciones](02-Migraciones%20y%20consultas.md))

**No tracking:** consulta cuyos resultados no entran al change tracker. ([Migraciones](02-Migraciones%20y%20consultas.md))

**ORM:** biblioteca que traduce entre objetos del programa y tablas relacionales. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Overposting:** modificación de propiedades que el caso de uso no pretendía exponer al cliente. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Post/Redirect/Get (PRG):** patrón que redirige a un GET después de procesar correctamente un POST. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Proveedor:** integración de EF Core para un motor concreto, como SQL Server, PostgreSQL o SQLite. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Proyección:** `Select` que devuelve solo los datos necesarios, normalmente en un DTO, en lugar de entidades completas. ([Migraciones](02-Migraciones%20y%20consultas.md))

**Snapshot del modelo:** representación usada para comparar el modelo actual con la última migración. ([Migraciones](02-Migraciones%20y%20consultas.md))

**Unidad de trabajo:** conjunto de cambios que se confirman juntos con `SaveChangesAsync`. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))
