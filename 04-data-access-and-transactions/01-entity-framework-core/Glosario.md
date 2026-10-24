# Glosario: Entity Framework Core

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Actualización masiva (bulk update):** `ExecuteUpdateAsync` o `ExecuteDeleteAsync`, que ejecutan un `UPDATE` o `DELETE` directo sin cargar entidades ni pasar por el change tracker. ([CRUD en APIs](06-CRUD%20en%20APIs%20con%20EF%20Core.md))

**ADO.NET:** conjunto de tipos de .NET (`DbConnection`, `DbCommand`, `DbDataReader`) para conectarse a una base y ejecutar SQL. ([ADO.NET y ORMs](01-ADO%20NET%20ORMs%20y%20EF%20Core.md))

**Antiforgery (CSRF):** protección que exige un token oculto en cada formulario para que otro sitio no pueda enviar peticiones en nombre del usuario. ([CRUD seguro](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Change tracker:** componente del `DbContext` que recuerda el estado (`Added`, `Unchanged`, `Modified`, `Deleted`) de cada entidad cargada. ([DbContext](02-DbContext%20entidades%20y%20configuracion.md))

**Clave foránea (FK):** columna cuyo valor debe existir como clave primaria en otra tabla. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Code First:** flujo donde el modelo de código y sus migraciones conducen la evolución del esquema. ([Migraciones](04-Migraciones%20y%20consultas.md))

**Concurrencia optimista:** estrategia que no bloquea al leer y detecta al guardar si otro proceso cambió la fila; EF Core lanza `DbUpdateConcurrencyException`. ([CRUD seguro](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Consulta parametrizada:** sentencia en la que los valores viajan aparte del texto SQL, como parámetros, y nunca se interpretan como código. ([ADO.NET y ORMs](01-ADO%20NET%20ORMs%20y%20EF%20Core.md))

**Conversión de valor (value converter):** regla que transforma un valor al guardarlo y al leerlo, como un enum a texto con `HasConversion<string>()`. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**Convención:** regla por defecto con la que EF Core decide el mapeo cuando no configuras nada. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**Data annotations:** atributos como `[Key]`, `[MaxLength]` o `[NotMapped]` que configuran el mapeo sobre la propia entidad. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**Datos semilla (seed data):** filas iniciales declaradas en el modelo con `HasData`, que se insertan al aplicar migraciones. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**DbContext:** sesión de trabajo de EF Core que rastrea entidades y coordina consultas y cambios. ([DbContext](02-DbContext%20entidades%20y%20configuracion.md))

**DbSet:** puerta de entrada tipada para consultar y modificar entidades de un tipo. ([DbContext](02-DbContext%20entidades%20y%20configuracion.md))

**DTO de entrada:** tipo que define exactamente qué datos acepta un endpoint; evita que el cliente fije campos del servidor. ([CRUD en APIs](06-CRUD%20en%20APIs%20con%20EF%20Core.md))

**Eager loading:** carga de datos relacionados en la misma consulta con `Include`/`ThenInclude`. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Entidad principal y dependiente:** la principal es la referenciada por la FK; la dependiente es la que contiene la FK. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Esquema (schema):** espacio de nombres dentro de una base (`dbo`, `ventas`) que agrupa tablas; las migraciones lo crean con `EnsureSchema`. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**Explicit loading:** carga de una navegación a pedido con `Entry(...).Collection(...).LoadAsync()`. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Explosión cartesiana:** multiplicación de filas al hacer `JOIN` con varias colecciones en una sola consulta. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Fluent API:** configuración del modelo con código en `OnModelCreating` o en clases `IEntityTypeConfiguration<T>`; gana sobre atributos y convenciones. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**Grafo de entidades:** una entidad junto con las entidades a las que apuntan sus navegaciones; `Add` agrega el grafo completo. ([CRUD en APIs](06-CRUD%20en%20APIs%20con%20EF%20Core.md))

**Inyección SQL (SQL injection):** ataque en el que un dato del usuario termina interpretado como parte de la sentencia SQL. ([ADO.NET y ORMs](01-ADO%20NET%20ORMs%20y%20EF%20Core.md))

**Lazy loading:** carga automática de una navegación la primera vez que se accede a ella; en EF Core requiere proxies y está apagado por defecto. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Micro-ORM:** biblioteca que solo convierte resultados de SQL escrito a mano en objetos, como Dapper. ([ADO.NET y ORMs](01-ADO%20NET%20ORMs%20y%20EF%20Core.md))

**Migración:** conjunto versionado de operaciones (`Up` y `Down`) para llevar el esquema de un estado a otro. ([Migraciones](04-Migraciones%20y%20consultas.md))

**No tracking:** consulta cuyos resultados no entran al change tracker. ([Migraciones](04-Migraciones%20y%20consultas.md))

**ORM (Object-Relational Mapper):** biblioteca que traduce entre objetos del programa y tablas relacionales. ([ADO.NET y ORMs](01-ADO%20NET%20ORMs%20y%20EF%20Core.md))

**Overposting:** modificación de propiedades que el caso de uso no pretendía exponer al cliente. ([CRUD seguro](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Post/Redirect/Get (PRG):** patrón que redirige a un GET después de procesar correctamente un POST. ([CRUD seguro](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md))

**Problema N+1:** una consulta para obtener una lista y una adicional por cada elemento al acceder a una navegación. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Propiedad de navegación:** propiedad que apunta a otra entidad o a una colección de entidades; no es una columna. ([Modelado](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md))

**Proveedor:** integración de EF Core para un motor concreto, como SQL Server, PostgreSQL o SQLite. ([DbContext](02-DbContext%20entidades%20y%20configuracion.md))

**Proveedor ADO.NET:** implementación de los tipos de ADO.NET para un motor (`Microsoft.Data.SqlClient`, `Microsoft.Data.Sqlite`, `Npgsql`). ([ADO.NET y ORMs](01-ADO%20NET%20ORMs%20y%20EF%20Core.md))

**Proyección:** `Select` que devuelve solo los datos necesarios, normalmente en un DTO, en lugar de entidades completas. ([Migraciones](04-Migraciones%20y%20consultas.md))

**Skip navigation:** navegación de muchos a muchos que salta la tabla intermedia, como `Libro.Autores`. ([Relaciones](05-Relaciones%20y%20carga%20de%20datos.md))

**Snapshot del modelo:** representación usada para comparar el modelo actual con la última migración. ([Migraciones](04-Migraciones%20y%20consultas.md))

**Unidad de trabajo:** conjunto de cambios que se confirman juntos con `SaveChangesAsync`. ([DbContext](02-DbContext%20entidades%20y%20configuracion.md))
