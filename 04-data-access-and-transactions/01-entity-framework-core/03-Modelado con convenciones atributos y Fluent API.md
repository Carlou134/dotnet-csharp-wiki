# Modelado con convenciones, atributos y Fluent API

## En una frase

EF Core construye el modelo en tres capas, **convenciones < atributos < Fluent API**, donde la última gana; conocer qué decide cada capa evita columnas que no querías, longitudes que no son las que crees y enums que cambian de significado.

-----

## Antes de empezar

Conviene que ya sepas:

* `DbContext`, `DbSet` y el change tracker: [DbContext, entidades y configuración](02-DbContext%20entidades%20y%20configuracion.md).
* Enums y tipos que aceptan `null`: [Tipos avanzados](../../01-csharp-core-and-runtime/05-tipos-avanzados/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Convención:** regla por defecto con la que EF Core decide el mapeo cuando no configuras nada.
* **Data annotations:** atributos como `[Key]` o `[MaxLength]` que se ponen sobre la entidad.
* **Propiedad de navegación:** propiedad que apunta a otra entidad (`Categoria`) o a una colección de ellas (`Tareas`); no es una columna.
* **Conversión de valor (*value converter*):** regla que transforma un valor al guardarlo y al leerlo, por ejemplo un enum a texto.
* **Esquema (*schema*):** espacio de nombres dentro de una base (`dbo`, `ventas`) que agrupa tablas.
* **Datos semilla (*seed data*):** filas iniciales que el modelo declara y que se insertan junto con el esquema.

-----

## El problema

Un equipo modela las tareas así:

```csharp
public enum Prioridad { Baja, Media, Alta }

public class Tarea
{
    public Guid TareaId { get; set; }

    [MaxLength(200)]
    public string Titulo { get; set; } = "";

    public Prioridad Prioridad { get; set; }

    public string Resumen { get; set; } = "";   // "se calcula en pantalla"
}

// En OnModelCreating, escrito meses después por otra persona:
modelBuilder.Entity<Tarea>().Property(t => t.Titulo).HasMaxLength(100);
```

Lo que pasa en producción:

1. Un usuario guarda un título de 150 caracteres. SQL Server responde `String or binary data would be truncated`. El atributo dice 200, pero la columna es `nvarchar(100)`: **la Fluent API gana** y nadie lo ve leyendo la entidad.
2. `Resumen` tiene `get` y `set`, así que EF Core crea una columna `Resumen` que nadie llena con sentido.
3. Alguien "ordena" el enum por importancia: `{ Alta, Media, Baja }`. EF Core guarda enums como `int`. Todas las tareas que tenían `0` (Baja) ahora se leen como **Alta**, sin un solo error.

Las consecuencias:

* reglas de esquema repartidas en dos lugares que se contradicen;
* columnas basura que confunden y ocupan espacio;
* datos que cambian de significado por un refactor inocente.

-----

## Cómo funciona

### 1. Tres capas y quién gana

```text
Convenciones ──► Data annotations ──► Fluent API
  (por defecto)     (sobre la clase)     (OnModelCreating / IEntityTypeConfiguration)
                                          ▲
                                          └── si dos capas configuran lo mismo, gana esta
```

Elige **una** fuente para cada regla. Mezclar atributo y Fluent API sobre la misma propiedad es exactamente el problema de arriba.

### 2. Lo que EF Core decide solo (convenciones)

| Convención | Ejemplo |
| --- | --- |
| Se mapean las propiedades públicas con `get` **y** `set` | `public string Resumen => ...` (solo `get`) no es columna |
| Clave primaria: `Id` o `<Clase>Id` | `TareaId` en `Tarea`; `PersonaId` en `Persona`. Un `ClienteId` dentro de `Pedido` **no** es la clave |
| Claves `int`/`long`: las genera la base | `IDENTITY` en SQL Server |
| Claves `Guid`: las genera EF Core al hacer `Add` | Sin tocar la base |
| Nulabilidad según *nullable reference types* | `string` → `NOT NULL`; `string?` → nullable |
| Nombre de tabla: el de la propiedad `DbSet`, si existe; si no, el de la clase | `DbSet<Tarea> Tareas` → tabla `Tareas`. **No pluraliza** nombres de clase |
| Relaciones: navegación + propiedad `<Navegación>Id` | `Categoria` + `CategoriaId` → FK |
| Enums como `int` | `Prioridad.Alta` → `2` |
| Strings sin longitud | `nvarchar(max)` en SQL Server |

Las convenciones resuelven la mayor parte del modelo. Lo que **nunca** deciden por ti son las reglas de negocio: longitudes, unicidad, precisión de decimales.

### 3. Data annotations

```csharp
[Table("Tarea", Schema = "ventas")]
[Index(nameof(Titulo))]
public class Tarea
{
    [Key]
    public Guid TareaId { get; set; }

    [Required, MaxLength(200)]
    public string Titulo { get; set; } = "";

    [NotMapped]
    public string Resumen { get; set; } = "";

    public Guid CategoriaId { get; set; }

    [ForeignKey(nameof(CategoriaId))]
    public Categoria? Categoria { get; set; }
}
```

| Atributo | Efecto en el esquema |
| --- | --- |
| `[Key]` | Clave primaria (útil cuando el nombre no sigue la convención) |
| `[Required]` | `NOT NULL` (con NRT activado, `string` ya lo es) |
| `[MaxLength(n)]` | `nvarchar(n)`; también lo usa la validación de ASP.NET Core |
| `[NotMapped]` | Sin columna |
| `[ForeignKey("X")]` | Indica qué propiedad es la FK de una navegación |
| `[Table]`, `[Column]` | Nombre de tabla, esquema, nombre o tipo de columna |
| `[Index]` (EF Core 5+) | Índice, opcionalmente único |
| `[PrimaryKey]` (EF Core 7+) | Clave compuesta sobre la clase |
| `[Precision(18, 2)]` | Precisión y escala de un `decimal` |

Ventaja: la regla está a la vista en la entidad. Costo: la entidad queda atada a detalles de persistencia, y algunas cosas no se pueden expresar.

### 4. Fluent API

```csharp
modelBuilder.Entity<Tarea>(tarea =>
{
    tarea.ToTable("Tarea");
    tarea.HasKey(t => t.TareaId);
    tarea.Property(t => t.Titulo).IsRequired().HasMaxLength(200);
    tarea.Property(t => t.Prioridad).HasConversion<string>().HasMaxLength(10);
    tarea.HasOne(t => t.Categoria).WithMany(c => c.Tareas).HasForeignKey(t => t.CategoriaId);
    tarea.Ignore(t => t.Resumen);
});
```

Se llama *fluent* porque cada método devuelve el *builder* y permite encadenar el siguiente. Hace todo lo que hacen los atributos y además cosas que no tienen atributo: conversiones de valor, datos semilla, comportamiento de borrado en cascada, valores por defecto con SQL (`HasDefaultValueSql`), filtros globales de consulta.

| | Atributos | Fluent API |
| --- | --- | --- |
| Dónde vive la regla | En la entidad | Fuera de la entidad |
| Alcance | Lo básico | Todo |
| Entidad independiente de EF Core | No | Sí |
| Validación de entrada gratis (`[MaxLength]`) | Sí (si la entidad se usa como modelo de entrada, que no deberías) | No |

Ninguna es "la profesional". Lo importante es que el equipo elija un criterio y no duplique reglas.

### 5. `IEntityTypeConfiguration<T>`: una clase por entidad

Con veinte entidades, `OnModelCreating` se vuelve un archivo de mil líneas. Separa la configuración:

```csharp
public sealed class TareaConfiguration : IEntityTypeConfiguration<Tarea>
{
    public void Configure(EntityTypeBuilder<Tarea> builder)
    {
        builder.ToTable("Tarea");
        builder.Property(t => t.Titulo).HasMaxLength(200);
    }
}

protected override void OnModelCreating(ModelBuilder modelBuilder) =>
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(TareasContext).Assembly);
```

`ApplyConfigurationsFromAssembly` busca **todas** las clases que implementan `IEntityTypeConfiguration<T>` en ese ensamblado, se llamen como se llamen.

### 6. Esquemas

```csharp
builder.ToTable("Tarea", "ventas");          // ventas.Tarea
modelBuilder.HasDefaultSchema("ventas");     // para todas las tablas sin esquema explícito
```

Al generar la migración, EF Core agrega `migrationBuilder.EnsureSchema(name: "ventas")` antes de crear la tabla: **no** necesitas crear el esquema a mano. Si no indicas esquema, EF Core no lo escribe en el SQL y SQL Server usa el esquema por defecto del usuario (normalmente `dbo`). SQLite no tiene esquemas y los ignora.

### 7. Claves `Guid`

```csharp
public Guid TareaId { get; set; }   // basta: EF Core asigna el valor en Add
```

* No hace falta `Guid.NewGuid()` en el constructor. Peor aún: EF Core también usa el constructor al **leer** filas, así que generarías un Guid que se descarta en cada materialización.
* Un Guid aleatorio (versión 4) como clave *clustered* en SQL Server fragmenta el índice, porque cada fila nueva cae en una posición al azar. El generador de EF Core para SQL Server produce valores secuenciales para evitarlo.
* Si generas el valor tú (por ejemplo, para devolverlo antes de guardar), .NET 9 agregó `Guid.CreateVersion7()`: incluye la marca de tiempo y se ordena por fecha de creación.

### 8. Enums: número o texto

| Estrategia | Columna | Riesgo |
| --- | --- | --- |
| Por defecto (`int`) | `0, 1, 2` | Reordenar o insertar valores cambia el significado de los datos |
| Valores explícitos (`Baja = 1, Media = 2, Alta = 3`) | `1, 2, 3` | Ninguno al reordenar; el número sigue sin decir nada a quien lee la tabla |
| `HasConversion<string>()` | `'Baja', 'Alta'` | Renombrar un miembro rompe la lectura; ocupa más espacio |

Elige una y sé explícito. Lo que **nunca** debes hacer es depender del orden de declaración.

### 9. Navegaciones: no son columnas

```csharp
public Guid CategoriaId { get; set; }            // columna (FK)
public Categoria? Categoria { get; set; }        // navegación: sin columna
public List<Tarea> Tareas { get; } = [];          // navegación de colección: sin columna
```

* SQL solo guarda valores escalares. La relación existe en la base gracias a la **FK**; las navegaciones existen solo en memoria para que recorras el grafo de objetos.
* Para colecciones sirve `ICollection<T>`, `List<T>`, `HashSet<T>`: EF Core acepta cualquiera que implemente `ICollection<T>`. `ICollection<T>` no es obligatorio ni "funciona mejor con proxies".
* `virtual` en una navegación solo importa si activas *lazy loading* con proxies. Lo verás en [Relaciones y carga de datos](05-Relaciones%20y%20carga%20de%20datos.md).

### 10. Datos semilla con `HasData`

```csharp
builder.HasData(
    new Categoria { CategoriaId = Guid.Parse("11111111-1111-1111-1111-111111111111"), Nombre = "Personal" },
    new Categoria { CategoriaId = Guid.Parse("22222222-2222-2222-2222-222222222222"), Nombre = "Trabajo" });
```

* La clave es obligatoria y fija, aunque normalmente se genere sola: EF Core compara semillas entre migraciones **por clave**.
* Al generar una migración aparecen `InsertData`, `UpdateData` o `DeleteData` según cambies, edites o quites semillas. Se aplican al aplicar la migración (y también con `EnsureCreated`).
* No uses `Guid.NewGuid()` ni `DateTime.Now` dentro de `HasData`: cada migración vería un valor distinto y generaría un `UpdateData` nuevo.
* Para datos que dependen del entorno o de otras consultas, EF Core 9 agregó `UseSeeding`/`UseAsyncSeeding` en las opciones del contexto.

-----

## Ejemplo completo

Consola con SQLite. `Modelado.csproj` igual al de la [lección 1](01-ADO%20NET%20ORMs%20y%20EF%20Core.md) (paquete `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0).

`Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

var options = new DbContextOptionsBuilder<TareasContext>()
    .UseSqlite("Data Source=modelado.db")
    .Options;

await using var db = new TareasContext(options);
await db.Database.EnsureDeletedAsync();
await db.Database.EnsureCreatedAsync();

// 1. Qué entendió EF Core del modelo
var tipo = db.Model.FindEntityType(typeof(Tarea))!;
Console.WriteLine($"Tabla: {tipo.GetTableName()}");
foreach (var nombre in new[] { "Titulo", "Descripcion", "CategoriaId" })
{
    var p = tipo.FindProperty(nombre)!;
    Console.WriteLine($"  {nombre,-12} max={p.GetMaxLength()?.ToString() ?? "-",-4} nullable={p.IsNullable}");
}
var prioridad = tipo.FindProperty(nameof(Tarea.Prioridad))!;
Console.WriteLine($"  {"Prioridad",-12} se guarda como {prioridad.GetValueConverter()!.ProviderClrType.Name}");
Console.WriteLine($"  {"Resumen",-12} mapeada={tipo.FindProperty("Resumen") is not null}");

// 2. Clave Guid y enum como texto
var personal = await db.Categorias.SingleAsync(c => c.Nombre == "Personal");
var tarea = new Tarea
{
    Titulo = "Pagar la luz",
    Prioridad = Prioridad.Alta,
    FechaCreacion = DateTime.UtcNow,
    CategoriaId = personal.CategoriaId
};
db.Tareas.Add(tarea);
Console.WriteLine($"Id generado al hacer Add: {tarea.TareaId != Guid.Empty}");
await db.SaveChangesAsync();

string guardado = await db.Database
    .SqlQuery<string>($"SELECT Prioridad AS Value FROM Tarea")
    .SingleAsync();
Console.WriteLine($"Prioridad en la columna: {guardado}");
Console.WriteLine($"Categorías sembradas: {await db.Categorias.CountAsync()}");

public enum Prioridad { Baja = 1, Media = 2, Alta = 3 }

public class Categoria
{
    public Guid CategoriaId { get; set; }
    public required string Nombre { get; set; }
    public string? Descripcion { get; set; }
    public List<Tarea> Tareas { get; } = [];
}

public class Tarea
{
    public Guid TareaId { get; set; }
    public required string Titulo { get; set; }
    public string? Descripcion { get; set; }
    public Prioridad Prioridad { get; set; }
    public DateTime FechaCreacion { get; set; }
    public Guid CategoriaId { get; set; }
    public Categoria? Categoria { get; set; }

    public string Resumen => $"{Titulo} ({Prioridad})";
}

public sealed class CategoriaConfiguration : IEntityTypeConfiguration<Categoria>
{
    public void Configure(EntityTypeBuilder<Categoria> builder)
    {
        builder.ToTable("Categoria");
        builder.Property(c => c.Nombre).HasMaxLength(150);
        builder.HasData(
            new Categoria { CategoriaId = Guid.Parse("11111111-1111-1111-1111-111111111111"), Nombre = "Personal" },
            new Categoria { CategoriaId = Guid.Parse("22222222-2222-2222-2222-222222222222"), Nombre = "Trabajo" });
    }
}

public sealed class TareaConfiguration : IEntityTypeConfiguration<Tarea>
{
    public void Configure(EntityTypeBuilder<Tarea> builder)
    {
        builder.ToTable("Tarea");
        builder.Property(t => t.Titulo).HasMaxLength(200);
        builder.Property(t => t.Prioridad).HasConversion<string>().HasMaxLength(10);
        builder.HasOne(t => t.Categoria)
            .WithMany(c => c.Tareas)
            .HasForeignKey(t => t.CategoriaId);
    }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Categoria> Categorias => Set<Categoria>();
    public DbSet<Tarea> Tareas => Set<Tarea>();

    protected override void OnModelCreating(ModelBuilder modelBuilder) =>
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(TareasContext).Assembly);
}
```

Salida:

```text
Tabla: Tarea
  Titulo       max=200  nullable=False
  Descripcion  max=-    nullable=True
  CategoriaId  max=-    nullable=False
  Prioridad    se guarda como String
  Resumen      mapeada=False
Id generado al hacer Add: True
Prioridad en la columna: Alta
Categorías sembradas: 2
```

Qué observar:

* La tabla se llama `Tarea` aunque el `DbSet` se llame `Tareas`: `ToTable` gana a la convención.
* `Titulo` es `NOT NULL` sin `[Required]`: lo decidió la convención de *nullable reference types*. `Descripcion` (`string?`) es nullable.
* `Resumen` no tiene columna y no hizo falta `Ignore`: solo tiene `get`.
* El Guid ya existe **antes** de `SaveChangesAsync`. Con una clave `int`, valdría 0 hasta guardar.
* La columna guarda el texto `Alta`. Reordenar el enum ya no corrompe nada.
* Las dos categorías existen sin ningún `Add`: vienen de `HasData`.

-----

## Errores comunes

**1. Configurar la misma propiedad con atributo y con Fluent API.**
Qué pasa: el esquema no coincide con lo que se lee en la entidad (`[MaxLength(200)]` y columna de 100).
Por qué: la Fluent API se aplica después y gana.
Arreglo: un solo lugar por regla. Si usas Fluent API, quita el atributo.

**2. Depender del orden del enum.**
Qué pasa: los datos guardados cambian de significado al reordenar o insertar un miembro.
Por qué: por defecto se guarda el número, y el número sale de la posición.
Arreglo: valores explícitos o `HasConversion<string>()`.

**3. Creer que hay que crear el esquema a mano antes de migrar.**
Qué pasa: scripts manuales innecesarios o pasos de despliegue que nadie recuerda.
Por qué: la migración ya incluye `EnsureSchema`.
Arreglo: declara el esquema en el modelo y deja que la migración lo cree.

**4. Generar el Guid en el constructor de la entidad.**
Qué pasa: un Guid nuevo cada vez que EF Core materializa una fila, que inmediatamente se sobrescribe con el valor de la base.
Por qué: EF Core usa el constructor también al leer.
Arreglo: deja que EF Core lo genere en `Add`, o asígnalo en un método de fábrica (`Tarea.Crear(...)`).

**5. Valores no deterministas en `HasData`.**
Qué pasa: cada `migrations add` genera un `UpdateData` aunque no cambiaste nada.
Por qué: `Guid.NewGuid()` o `DateTime.Now` dan un valor distinto cada vez que se construye el modelo.
Arreglo: constantes (`Guid.Parse("...")`, `new DateTime(2026, 1, 1)`).

**6. Usar el proveedor InMemory para "probar la base".**
Qué pasa: las pruebas pasan y producción falla: no hay FKs, ni transacciones reales, ni traducción a SQL.
Por qué: InMemory no es una base relacional; es una colección en memoria.
Arreglo: SQLite en memoria para pruebas rápidas, o el motor real en un contenedor para pruebas de integración.

-----

## Según la versión de .NET

* **EF Core 2.1:** conversiones de valor (`HasConversion`) y datos semilla con `HasData`.
* **EF Core 5:** atributo `[Index]`.
* **EF Core 6:** configuración previa a convenciones (`ConfigureConventions`): por ejemplo, todos los `string` con `HaveMaxLength(500)` o todos los enums como texto en una línea.
* **EF Core 7:** atributos `[PrimaryKey]` y `[DeleteBehavior]`.
* **EF Core 8:** *complex types* para agrupar columnas en un objeto sin identidad propia.
* **EF Core 9:** `UseSeeding`/`UseAsyncSeeding`; .NET 9 agrega `Guid.CreateVersion7()`.
* **EF Core 10:** acompaña a .NET 10 (LTS).

-----

## Cuándo sí y cuándo no

**Usa convenciones cuando:** el nombre y el tipo ya expresan lo que quieres. No configures lo que EF Core ya deduce.

**Usa atributos cuando:** el proyecto es chico, las reglas son simples y prefieres verlas en la entidad.

**Usa Fluent API con `IEntityTypeConfiguration` cuando:** quieres entidades sin dependencias de EF Core (por ejemplo, un dominio rico), necesitas conversiones, semillas o relaciones con comportamiento específico, o el modelo tiene muchas entidades.

**No uses `HasData` cuando:** los datos dependen del entorno, se generan en tiempo de ejecución o son muchos: usa `UseSeeding` o un proceso de carga aparte.

-----

## Resumen en 5 líneas

1. Convenciones, atributos y Fluent API se combinan; si chocan, gana la Fluent API.
2. Las convenciones mapean propiedades con `get`/`set`, claves `Id`/`<Clase>Id` y FKs por nombre; no pluralizan.
3. Las navegaciones no son columnas: la relación vive en la FK.
4. Guarda enums con valores explícitos o como texto, nunca dependiendo del orden.
5. `HasData` necesita claves fijas y valores deterministas; las migraciones crean el esquema solas.

-----

## Para profundizar

<details>
<summary>Convenciones globales con <code>ConfigureConventions</code></summary>

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<string>().HaveMaxLength(500);
    configurationBuilder.Properties<decimal>().HavePrecision(18, 2);
    configurationBuilder.Properties<Enum>().HaveConversion<string>();
}
```

Define la regla una vez para todo el modelo. Cada entidad puede sobrescribirla con su propia configuración. Evita olvidar `HasMaxLength` en la propiedad número cuarenta.

</details>

<details>
<summary>Ver el modelo completo que construyó EF Core</summary>

`Console.WriteLine(db.Model.ToDebugString());` imprime entidades, propiedades, claves, índices y relaciones tal como EF Core las entendió. Es la forma más rápida de responder "¿por qué me creó esta columna?" sin generar una migración.

</details>

-----

## En entrevista

### Respuesta corta (junior)

EF Core mapea por convención: propiedades públicas a columnas, `Id` o `<Clase>Id` como clave, y FKs por nombre. Para lo que no deduce uso atributos como `[MaxLength]` o la Fluent API en `OnModelCreating`. Si ambos configuran lo mismo, gana la Fluent API.

### Respuesta ampliada (semi-senior)

Uso convenciones para lo obvio y Fluent API en clases `IEntityTypeConfiguration<T>` para lo demás, registradas con `ApplyConfigurationsFromAssembly`, así la entidad no depende de EF Core. Defino longitudes, precisión e índices únicos explícitamente, guardo enums como texto o con valores fijos y configuro convenciones globales con `ConfigureConventions`. Para semillas uso `HasData` con claves constantes, o `UseSeeding` cuando dependen del entorno. Sé que las navegaciones no son columnas y que `virtual` solo importa con lazy loading.

### Preguntas frecuentes de seguimiento

**1. ¿Qué gana, `[MaxLength(200)]` o `HasMaxLength(100)`?**
`HasMaxLength(100)`: la Fluent API se aplica después.

**2. ¿EF Core pluraliza los nombres de tabla?**
No. Usa el nombre del `DbSet` o, si no hay, el de la clase.

**3. ¿Por qué `HasData` exige la clave?**
Porque compara semillas entre migraciones por clave para decidir si insertar, actualizar o borrar.

**4. ¿Hay que crear el esquema antes de la migración?**
No. La migración incluye `EnsureSchema`.

-----

## Práctica

**Ejercicio 1.** Esta entidad tiene al menos cuatro problemas de mapeo. Encuéntralos y corrígelos con Fluent API.

```csharp
public class Usuario
{
    public int Id { get; set; }
    public string Email { get; set; } = "";
    public string PasswordConfirmacion { get; set; } = "";
    public string NombreCompleto { get; set; } = "";
    public string Iniciales { get; set; } = "";
    public Rol Rol { get; set; }
}
```

<details>
<summary>Solución</summary>

* `PasswordConfirmacion` e `Iniciales` tienen `get`/`set`: se mapean. Una es dato del formulario, la otra se calcula.
* `Email` y `NombreCompleto` quedan `nvarchar(max)` sin unicidad.
* `Rol` se guarda como `int` dependiente del orden.

```csharp
public class Usuario
{
    public int Id { get; set; }
    public required string Email { get; set; }
    public required string NombreCompleto { get; set; }
    public Rol Rol { get; set; }
    public string Iniciales => string.Concat(NombreCompleto.Split(' ', StringSplitOptions.RemoveEmptyEntries).Select(p => p[0]));
}

public sealed class UsuarioConfiguration : IEntityTypeConfiguration<Usuario>
{
    public void Configure(EntityTypeBuilder<Usuario> builder)
    {
        builder.Property(u => u.Email).HasMaxLength(254);
        builder.HasIndex(u => u.Email).IsUnique();
        builder.Property(u => u.NombreCompleto).HasMaxLength(150);
        builder.Property(u => u.Rol).HasConversion<string>().HasMaxLength(20);
    }
}
```

`PasswordConfirmacion` sale de la entidad: pertenece al modelo de entrada del formulario, no a la tabla. `Iniciales` pasa a ser de solo lectura y deja de mapearse sin `Ignore`.

</details>

**Ejercicio 2.** Agrega una tercera categoría semilla, "Hogar", y explica qué verás en la próxima migración.

<details>
<summary>Solución</summary>

```csharp
new Categoria { CategoriaId = Guid.Parse("33333333-3333-3333-3333-333333333333"), Nombre = "Hogar" }
```

La migración contendrá un único `migrationBuilder.InsertData(table: "Categoria", ...)` con esa fila, y su `Down` un `DeleteData` con la misma clave. Las dos categorías anteriores no aparecen porque sus claves y valores no cambiaron.

</details>

-----

## Siguiente lección

[Migraciones y consultas](04-Migraciones%20y%20consultas.md)
