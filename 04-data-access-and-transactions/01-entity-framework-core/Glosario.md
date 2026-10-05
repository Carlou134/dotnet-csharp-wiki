# Glosario

-----

**Code First:** flujo donde el modelo de código y sus migraciones conducen la evolución del esquema. ([Migraciones](02-Migraciones%20y%20consultas.md))

**Concurrencia optimista:** estrategia que no bloquea al leer y detecta al guardar si otro proceso cambió la fila; EF Core lanza `DbUpdateConcurrencyException`. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**DbContext:** sesión de trabajo de EF Core que rastrea entidades y coordina consultas y cambios. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**DbSet:** puerta de entrada tipada para consultar y modificar entidades de un tipo. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Migración:** conjunto versionado de operaciones para llevar el esquema de un estado a otro. ([Migraciones](02-Migraciones%20y%20consultas.md))

**No tracking:** consulta que no conserva entidades en el change tracker. ([Migraciones](02-Migraciones%20y%20consultas.md))

**Overposting:** modificación de propiedades que el caso de uso no pretendía exponer al cliente. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Post/Redirect/Get (PRG):** patrón que redirige a un GET después de procesar correctamente un POST. ([CRUD seguro](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Proveedor:** integración de EF Core para un motor concreto, como SQL Server, PostgreSQL o SQLite. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))

**Snapshot del modelo:** representación usada para comparar el modelo actual con la última migración. ([Migraciones](02-Migraciones%20y%20consultas.md))

**Unidad de trabajo:** límite que agrupa cambios y confirma su persistencia. ([DbContext](01-DbContext%20entidades%20y%20configuracion.md))
