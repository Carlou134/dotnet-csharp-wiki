# DbContext, entidades y configuración

## En una frase

`DbContext` representa una sesión corta con la base de datos: construye el modelo a partir de tus entidades, traduce LINQ a SQL, **rastrea** los cambios de los objetos que carga y los confirma juntos al llamar a `SaveChangesAsync`.

-----

## Antes de empezar

Conviene que ya sepas:

* Inyección de dependencias y lifetimes: [Host, configuración y pipeline HTTP](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* LINQ: [Introducción a LINQ](../../01-csharp-core-and-runtime/07-linq/01-Introduccion%20a%20LINQ.md).
* `async`/`await` y `CancellationToken`: [Asincronía](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **ORM:** biblioteca que traduce entre objetos y tablas relacionales.
* **Proveedor:** paquete que adapta EF Core a un motor concreto (SQLite, SQL Server, PostgreSQL).
* **DbSet:** punto de entrada tipado para consultar y modificar entidades de un tipo.
* **Change tracker:** componente del `DbContext` que recuerda el estado de cada entidad cargada.
* **Unidad de trabajo:** conjunto de cambios que se confirman juntos.
* **Fluent API:** configuración del modelo con código en `OnModelCreating`.

-----

## El problema

Para "ganar rendimiento", un servicio carga pedidos y clientes en paralelo con el mismo contexto:

```csharp
var pedidosTask = db.Pedidos.ToListAsync();
var clientesTask = db.Clientes.ToListAsync();
await Task.WhenAll(pedidosTask, clientesTask);
```

En ejecución:

```text
InvalidOperationException: A second operation was started on this context instance before a previous
operation completed. This is usually caused by different threads concurrently using the same instance
of DbContext.
```

Otro equipo registró el contexto como singleton "para no crearlo en cada petición". Al principio funciona. Con carga real aparecen el mismo error de forma intermitente, consumo de memoria creciente y datos viejos: el change tracker acumula cada entidad que cualquier petición cargó desde que arrancó la aplicación.

Y la entidad se declaró sin configuración:

```csharp
public class Contacto
{
    public int Id { get; set; }
    public string Email { get; set; }
}
```

En SQL Server, `Email` termina como `nvarchar(max)`, sin índice y sin unicidad. Dos contactos con el mismo email entran sin problema.

Las consecuencias:

* errores de concurrencia que solo aparecen bajo carga;
* memoria que crece porque un contexto de larga vida nunca suelta entidades;
* un esquema que no expresa las reglas: longitudes, unicidad, obligatoriedad.

-----

## Cómo funciona

### 1. Proveedor: EF Core habla con un motor concreto

EF Core no se conecta "a cualquier base". Cada motor necesita su paquete:

| Motor | Paquete | Comentario |
| --- | --- | --- |
| SQLite | `Microsoft.EntityFrameworkCore.Sqlite` | Un archivo; portable; ideal para aprender |
| SQL Server | `Microsoft.EntityFrameworkCore.SqlServer` | LocalDB es una variante de desarrollo solo para Windows |
| PostgreSQL | `Npgsql.EntityFrameworkCore.PostgreSQL` | Proveedor de la comunidad Npgsql |

Mantén la **misma versión principal** en todos los paquetes de EF Core del proyecto.

### 2. Entidad

```csharp
public sealed class Contact
{
    public int Id { get; private set; }
    public required string Name { get; set; }
    public required string Email { get; set; }
}
```

* La entidad no hereda de nada de EF Core.
* `Id` de tipo `int` es clave primaria por convención, y la base genera su valor.
* Con *nullable reference types* activado, `string` se mapea como `NOT NULL` y `string?` como nullable.
* `private set` en `Id` impide que el código lo cambie; EF Core igual puede asignarlo.

### 3. Contexto

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Contact> Contacts => Set<Contact>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Contact>(contact =>
        {
            contact.Property(c => c.Name).HasMaxLength(120);
            contact.Property(c => c.Email).HasMaxLength(254);
            contact.HasIndex(c => c.Email).IsUnique();
        });
    }
}
```

Hay tres formas de configurar el modelo, y se combinan:

| Forma | Ejemplo | Cuándo |
| --- | --- | --- |
| Convenciones | `Id` es clave | Siempre, como base |
| Atributos | `[MaxLength(120)]` | Reglas simples que también sirven para validar |
| Fluent API | `HasIndex(...).IsUnique()` | Índices, relaciones, todo lo que los atributos no expresan |

En proyectos grandes, separa la configuración en clases `IEntityTypeConfiguration<T>` y regístralas con `modelBuilder.ApplyConfigurationsFromAssembly(...)`.

### 4. El change tracker y los estados

```text
                 Add()                 SaveChanges()
   (nuevo) ───────────────► Added ──────────────────► Unchanged
                                                        │  cambias una propiedad
                                                        ▼
                             Modified ◄─────────────────┘
                                │ SaveChanges()
                                ▼
                            Unchanged
   Remove() ───► Deleted ──SaveChanges()──► (Detached: ya no existe)
```

`Add`, `Update` y `Remove` **no ejecutan SQL**: cambian el estado en memoria. `SaveChangesAsync` recorre el change tracker, genera los `INSERT`/`UPDATE`/`DELETE` y los envía en una transacción.

### 5. Vida corta: scoped por petición

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("Database")));
```

`AddDbContext` registra el contexto como **scoped**: una instancia por petición HTTP, que se descarta al terminar. Así:

* cada petición tiene su propio change tracker, que no crece sin límite;
* nunca hay dos hilos usando el mismo contexto a la vez.

`DbContext` **no es seguro entre hilos**. Dentro de una petición, ejecuta las consultas una después de otra (`await` a cada una). Si de verdad necesitas paralelismo, usa un contexto por tarea con `IDbContextFactory<T>`.

### 6. Cadena de conexión fuera del código

```json
{
  "ConnectionStrings": {
    "Database": "Data Source=contacts.db"
  }
}
```

Una ruta de SQLite no es secreta. Una cadena con usuario y contraseña sí lo es: va en *user secrets* durante el desarrollo y en variables de entorno o un gestor de secretos en producción.

-----

## Ejemplo completo

Aplicación de consola. `Contacts.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseSqlite("Data Source=contacts.db")
    .Options;

await using var db = new AppDbContext(options);

// Solo para el ejemplo: recrea la base. En una aplicación real se usan migraciones.
await db.Database.EnsureDeletedAsync();
await db.Database.EnsureCreatedAsync();

var contacto = new Contact { Name = "Ana", Email = "ana@example.com" };
db.Contacts.Add(contacto);
Console.WriteLine($"Tras Add:          {db.Entry(contacto).State}, Id = {contacto.Id}");

await db.SaveChangesAsync();
Console.WriteLine($"Tras SaveChanges:  {db.Entry(contacto).State}, Id = {contacto.Id}");

contacto.Email = "ana@empresa.com";
Console.WriteLine($"Tras modificar:    {db.Entry(contacto).State}");

await db.SaveChangesAsync();
Console.WriteLine($"Tras guardar otra: {db.Entry(contacto).State}");

Console.WriteLine($"Contactos en la base: {await db.Contacts.CountAsync()}");

public sealed class Contact
{
    public int Id { get; private set; }
    public required string Name { get; set; }
    public required string Email { get; set; }
}

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Contact> Contacts => Set<Contact>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Contact>(contact =>
        {
            contact.Property(c => c.Name).HasMaxLength(120);
            contact.Property(c => c.Email).HasMaxLength(254);
            contact.HasIndex(c => c.Email).IsUnique();
        });
    }
}
```

Salida:

```text
Tras Add:          Added, Id = 0
Tras SaveChanges:  Unchanged, Id = 1
Tras modificar:    Modified
Tras guardar otra: Unchanged
Contactos en la base: 1
```

Qué observar:

* Después de `Add`, el contacto **no está en la base**: su estado es `Added` y su `Id` sigue en 0. El `INSERT` ocurre en `SaveChangesAsync`, que además trae el `Id` generado.
* Cambiar una propiedad no ejecuta SQL: el change tracker detecta la diferencia y marca `Modified`.
* El segundo `SaveChangesAsync` envía un `UPDATE` solo de la columna `Email`.
* `EnsureCreatedAsync` crea el esquema sin migraciones. Sirve para ejemplos y pruebas; no se combina con migraciones.

-----

## Errores comunes

**1. Registrar el contexto como singleton.**
Qué pasa: errores intermitentes de concurrencia, memoria que crece y datos desactualizados.
Por qué: todas las peticiones comparten un change tracker que nunca se vacía, y `DbContext` no es seguro entre hilos.
Arreglo: `AddDbContext` (scoped). En servicios singleton o en segundo plano, usa `IDbContextFactory<T>`.

**2. Lanzar consultas en paralelo sobre el mismo contexto.**
Qué pasa: `InvalidOperationException: A second operation was started on this context instance...`.
Por qué: un contexto admite una operación a la vez.
Arreglo: `await` a cada consulta en secuencia, o un contexto por tarea con `IDbContextFactory<T>`.

**3. Dejar strings sin longitud ni índices.**
Qué pasa: columnas `nvarchar(max)` en SQL Server, sin unicidad, con duplicados que el negocio no admite.
Por qué: sin configuración, EF Core no inventa reglas.
Arreglo: `HasMaxLength`, `HasIndex(...).IsUnique()` y nulabilidad correcta en las propiedades.

**4. Guardar credenciales en `appsettings.json`.**
Qué pasa: la contraseña queda en Git, incluso después de borrar la línea.
Por qué: el historial conserva cada versión del archivo.
Arreglo: rota la credencial y usa user secrets o variables de entorno.

**5. Creer que LocalDB es SQLite.**
Qué pasa: el proyecto no funciona en Linux ni en contenedores.
Por qué: LocalDB es SQL Server para desarrollo en Windows; SQLite es otro motor.
Arreglo: elige el proveedor según dónde corre la aplicación y configúralo explícitamente.

-----

## Según la versión de .NET

* **EF6 (Entity Framework 6):** generación clásica para .NET Framework; no es EF Core.
* **EF Core 1.0 (2016):** reescritura para .NET moderno.
* **EF Core 3.0:** las consultas que no se pueden traducir a SQL lanzan excepción en vez de evaluarse en memoria.
* **EF Core 7:** `ExecuteUpdate` y `ExecuteDelete` para cambios masivos sin cargar entidades.
* **EF Core 8:** *complex types* y colecciones de tipos primitivos.
* **EF Core 10:** acompaña a .NET 10 (LTS); agrega los operadores LINQ `LeftJoin`/`RightJoin` y filtros de consulta con nombre.

-----

## Cuándo sí y cuándo no

**Usa EF Core cuando:** tu modelo es relacional, la mayoría de las operaciones son CRUD y consultas expresables en LINQ, y quieres migraciones versionadas.

**No lo uses a ciegas cuando:** necesitas SQL muy específico, procesamiento masivo o control fino del plan de ejecución. Puedes combinarlo con SQL explícito (`FromSql`, `ExecuteUpdate`) o con Dapper en los puntos críticos, siempre midiendo.

-----

## Resumen en 5 líneas

1. `DbContext` es una sesión corta: scoped por petición y nunca compartida entre hilos.
2. Cada motor necesita su proveedor, con la misma versión principal que EF Core.
3. El change tracker registra estados; `Add` y la modificación de propiedades no ejecutan SQL.
4. `SaveChangesAsync` confirma la unidad de trabajo en una transacción.
5. Longitudes, índices y nulabilidad se configuran: EF Core no adivina las reglas.

-----

## Para profundizar

<details>
<summary>¿Repositorio sobre EF Core?</summary>

`DbContext` ya implementa unidad de trabajo y `DbSet<T>` se parece a un repositorio. Una abstracción adicional vale la pena cuando expresa el lenguaje del dominio (`IPedidoRepository.ObtenerPendientesDeEnvio()`) o un límite de arquitectura. Envolver cada método de EF Core sin aportar significado solo agrega código y oculta capacidades como `AsNoTracking` o `Include`.

</details>

<details>
<summary>DbContext pooling</summary>

`AddDbContextPool<AppDbContext>(...)` reutiliza instancias de contexto entre peticiones, reseteando su estado. Reduce el costo de crear contextos en aplicaciones con mucho tráfico. Tiene una condición: el contexto no puede guardar estado propio en campos, porque sobreviviría entre peticiones.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`DbContext` conecta el modelo de objetos con la base de datos: ejecuta consultas, rastrea cambios y los guarda con `SaveChanges`. Se registra como scoped, una instancia por petición, porque no es seguro entre hilos.

### Respuesta ampliada (semi-senior)

`DbContext` es una unidad de trabajo de vida corta. El change tracker marca estados (`Added`, `Modified`, `Deleted`) y `SaveChangesAsync` genera el SQL en una transacción. Lo registro scoped y uso `IDbContextFactory` en servicios singleton o en segundo plano. Configuro el modelo con Fluent API (longitudes, índices únicos, nulabilidad), separado en `IEntityTypeConfiguration`. Elijo el proveedor según el entorno de despliegue y saco las credenciales del código.

### Preguntas frecuentes de seguimiento

**1. ¿`DbSet` es una tabla?**
Es una abstracción de consultas y cambios; el mapeo decide a qué tabla corresponde.

**2. ¿`Add` ejecuta un INSERT?**
No. Marca la entidad como `Added`; el `INSERT` ocurre en `SaveChanges`.

**3. ¿Por qué no un DbContext singleton?**
Porque no es seguro entre hilos y su change tracker crecería sin límite.

**4. ¿Atributos o Fluent API?**
Ambos. La Fluent API expresa más (índices, relaciones) y mantiene la entidad libre de detalles de persistencia.

-----

## Práctica

**Ejercicio 1.** Cambia el ejemplo para usar SQL Server sin tocar la entidad. ¿Qué cambia y qué debes revisar?

<details>
<summary>Solución</summary>

Agrega el paquete `Microsoft.EntityFrameworkCore.SqlServer` (misma versión principal), reemplaza `UseSqlite(...)` por `UseSqlServer(...)` y lee la cadena desde configuración segura. La entidad y el contexto no cambian. Debes revisar las migraciones: se generan para un proveedor concreto, y los tipos de columna (`nvarchar(120)` frente a `TEXT`) son distintos.

</details>

**Ejercicio 2.** Un servicio en segundo plano (`BackgroundService`, que es singleton) debe leer contactos cada minuto. ¿Cómo obtiene un `DbContext` sin caer en el error del problema?

<details>
<summary>Solución</summary>

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(o => o.UseSqlite(connection));

public sealed class SincronizadorContactos(IDbContextFactory<AppDbContext> factory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var db = await factory.CreateDbContextAsync(stoppingToken);
            var total = await db.Contacts.CountAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

Cada iteración crea un contexto nuevo y lo descarta con `await using`. El change tracker no crece y no hay contexto compartido entre hilos.

</details>

-----

## Siguiente lección

[Migraciones y consultas](02-Migraciones%20y%20consultas.md)
