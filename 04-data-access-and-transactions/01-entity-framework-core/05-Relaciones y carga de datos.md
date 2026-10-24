# Relaciones y carga de datos

## En una frase

Una relación vive en la base como una **clave foránea** y en C# como **propiedades de navegación**; EF Core no carga las navegaciones por su cuenta, así que tú decides cuándo y cómo traer los datos relacionados, y esa decisión define cuántas consultas viajan a la base.

-----

## Antes de empezar

Conviene que ya sepas:

* Navegaciones, FKs por convención y Fluent API: [Modelado con convenciones, atributos y Fluent API](03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md).
* Consultas, proyecciones y `AsNoTracking`: [Migraciones y consultas](04-Migraciones%20y%20consultas.md).
* `Join` y `GroupBy` en LINQ: [LINQ](../../01-csharp-core-and-runtime/07-linq/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Clave foránea (FK):** columna cuyo valor debe existir como clave primaria en otra tabla.
* **Entidad principal y dependiente:** la principal es la referenciada (`Categoria`); la dependiente es la que tiene la FK (`Tarea`).
* **Eager loading:** cargar los datos relacionados en la misma consulta con `Include`.
* **Lazy loading:** cargar una navegación automáticamente la primera vez que se accede a ella.
* **Explicit loading:** cargar una navegación a pedido con `Entry(...).Collection(...).LoadAsync()`.
* **Problema N+1:** una consulta para la lista y una más por cada elemento.
* **Explosión cartesiana:** filas duplicadas que aparecen al hacer `JOIN` con varias colecciones a la vez.
* **Skip navigation:** navegación de muchos a muchos que salta la tabla intermedia (`Libro.Autores`).

-----

## El problema

Un endpoint lista categorías con la cantidad de tareas de cada una:

```csharp
var categorias = await db.Categorias.ToListAsync();

foreach (var c in categorias)
{
    resultado.Add(new { c.Nombre, Total = c.Tareas.Count });
}
```

Primera versión: devuelve `Total = 0` en todas. `Tareas` está vacía porque **EF Core no carga navegaciones si no se lo pides**.

Alguien lo "arregla" activando *lazy loading*. Ahora los totales son correctos, pero con 200 categorías el endpoint tarda dos segundos: el log muestra **201 consultas**, una para la lista y una por cada `c.Tareas`.

Otra persona lo reescribe con `Include` para tres colecciones:

```csharp
db.Categorias
    .Include(c => c.Tareas)
    .Include(c => c.Responsables)
    .Include(c => c.Etiquetas)
```

Una sola consulta, sí. Pero con 50 tareas, 5 responsables y 10 etiquetas por categoría, el `JOIN` devuelve 50 × 5 × 10 = **2500 filas por categoría** para armar 65 objetos.

Las consecuencias:

* navegaciones vacías que parecen datos reales ("0 tareas");
* N+1 consultas que crecen con los datos y nadie ve en desarrollo;
* consultas únicas que mueven miles de filas duplicadas.

-----

## Cómo funciona

### 1. Uno a muchos

![Relación uno a muchos](../../Images/Relacion%201-muchos.png)

```csharp
public class Categoria
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public List<Tarea> Tareas { get; } = [];          // lado "muchos" visto desde el principal
}

public class Tarea
{
    public int Id { get; set; }
    public string Titulo { get; set; } = "";
    public int CategoriaId { get; set; }               // FK: la única columna de la relación
    public Categoria? Categoria { get; set; }          // navegación hacia el principal
}
```

```text
Categoria (principal)               Tarea (dependiente)
┌────┬──────────┐                   ┌────┬──────────────┬─────────────┐
│ Id │ Nombre   │◄──────────────────│ Id │ Titulo       │ CategoriaId │
├────┼──────────┤        FK         ├────┼──────────────┼─────────────┤
│ 1  │ Hogar    │                   │ 10 │ Pagar luz    │ 1           │
└────┴──────────┘                   │ 11 │ Barrer       │ 1           │
                                    └────┴──────────────┴─────────────┘
```

EF Core detecta la relación por convención. Con Fluent API, la misma relación es:

```csharp
modelBuilder.Entity<Tarea>()
    .HasOne(t => t.Categoria)
    .WithMany(c => c.Tareas)
    .HasForeignKey(t => t.CategoriaId);
```

Como `CategoriaId` es `int` (no admite `null`), la relación es **obligatoria** y el borrado es en **cascada** por defecto: borrar la categoría borra sus tareas. Con `int?`, sería opcional y EF Core pondría la FK en `null`.

### 2. Uno a uno

![Relación uno a uno](../../Images/Relacion%201-1.png)

```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; } = "";
    public PrecioPromocion? PrecioPromocion { get; set; }
}

public class PrecioPromocion
{
    public int Id { get; set; }
    public decimal Precio { get; set; }
    public int LibroId { get; set; }          // la FK decide quién es el dependiente
    public Libro? Libro { get; set; }
}
```

```csharp
modelBuilder.Entity<Libro>()
    .HasOne(l => l.PrecioPromocion)
    .WithOne(p => p.Libro)
    .HasForeignKey<PrecioPromocion>(p => p.LibroId);
```

En uno a uno, `HasForeignKey<T>` necesita el tipo porque cualquiera de los dos lados podría tener la FK. EF Core crea además un **índice único** sobre `LibroId`: sin él, nada impediría dos precios para el mismo libro.

### 3. Muchos a muchos

![Relación muchos a muchos con tabla intermedia](../../Images/Relacion%20muchos-muchos.png)

Desde EF Core 5 no necesitas escribir la tabla intermedia:

```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; } = "";
    public List<Autor> Autores { get; } = [];
}

public class Autor
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public List<Libro> Libros { get; } = [];
}
```

EF Core crea la tabla `AutorLibro` con las columnas `AutoresId` y `LibrosId` y una clave compuesta. En el código navegas directo: `libro.Autores`.

Si la relación tiene **datos propios** (el orden del autor en la portada), la tabla intermedia es una entidad de verdad:

```csharp
public class LibroAutor
{
    public int LibroId { get; set; }
    public Libro? Libro { get; set; }
    public int AutorId { get; set; }
    public Autor? Autor { get; set; }
    public int Orden { get; set; }
}

modelBuilder.Entity<LibroAutor>().HasKey(la => new { la.LibroId, la.AutorId });
```

| | Skip navigations | Entidad intermedia explícita |
| --- | --- | --- |
| Código | `libro.Autores` | `libro.LibroAutores.Select(la => la.Autor)` |
| Datos en la relación | No | Sí (`Orden`, `FechaAlta`) |
| Cuándo | La relación es solo "está vinculado" | La relación tiene atributos o reglas |

### 4. Formas de cargar datos relacionados

| Forma | Código | Consultas | Riesgo |
| --- | --- | --- | --- |
| Eager | `Include(c => c.Tareas)` | 1 (con `JOIN`) | Explosión cartesiana con varias colecciones |
| Eager dividida | `Include(...).AsSplitQuery()` | 1 por colección | Datos inconsistentes si alguien escribe entre consultas |
| Explicit | `db.Entry(c).Collection(c => c.Tareas).LoadAsync()` | 1 por llamada | N+1 si la llamas en un bucle |
| Lazy | Acceso a `c.Tareas` con proxies | 1 por navegación accedida | N+1 invisible; consultas síncronas |
| Proyección | `Select(c => new { c.Nombre, c.Tareas.Count })` | 1 | Ninguno relevante: trae solo lo pedido |

```csharp
// Navegación anidada
db.Libros.Include(l => l.LibroAutores).ThenInclude(la => la.Autor);

// Include filtrado y ordenado (EF Core 5+)
db.Categorias.Include(c => c.Tareas.Where(t => !t.Completada).OrderBy(t => t.Titulo));
```

### 5. Lazy loading: apagado por defecto, por una buena razón

En EF Core **no** está activo de fábrica. Para activarlo necesitas:

1. el paquete `Microsoft.EntityFrameworkCore.Proxies`;
2. `options.UseLazyLoadingProxies()`;
3. entidades públicas, no `sealed`, con navegaciones `virtual`.

EF Core genera en tiempo de ejecución una subclase de tu entidad que intercepta el `get` de cada navegación y lanza una consulta. Por eso existe la palabra `virtual` en tantos tutoriales: sin ella no hay nada que interceptar.

```text
foreach (var c in categorias)       consulta 1:  SELECT ... FROM Categorias
    c.Tareas.Count                  consulta 2:  SELECT ... FROM Tareas WHERE CategoriaId = 1
                                    consulta 3:  SELECT ... FROM Tareas WHERE CategoriaId = 2
                                    ...
                                    consulta N+1
```

Además, esas consultas son **síncronas** (ocurren dentro de un `get`), así que bloquean el hilo aunque el resto del código sea `async`. Y si serializas una entidad con proxies a JSON, el serializador recorre todas las navegaciones y dispara una consulta por cada una.

### 6. `Include` con varias colecciones: `AsSplitQuery`

```text
Una consulta con 2 colecciones:            Consultas divididas:
Categoria × Tareas × Etiquetas             SELECT ... FROM Categorias
  1 × 50 × 10 = 500 filas por categoría    SELECT ... FROM Tareas    JOIN Categorias ...
                                           SELECT ... FROM Etiquetas JOIN Categorias ...
                                           50 + 10 filas por categoría
```

`AsSplitQuery()` evita la explosión a cambio de más idas y vueltas. Puedes ponerlo por defecto con `UseSqlServer(..., o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery))` y volver a una sola consulta con `AsSingleQuery()` donde convenga.

### 7. Para una API, proyecta

Si el resultado es un DTO, casi nunca necesitas `Include`:

```csharp
var resumen = await db.Categorias
    .OrderBy(c => c.Nombre)
    .Select(c => new CategoriaResumen(c.Nombre, c.Tareas.Count))
    .ToListAsync(ct);
```

EF Core traduce `c.Tareas.Count` a un `COUNT` en SQL. No carga ninguna tarea, no rastrea nada y no hay riesgo de ciclos al serializar.

-----

## Ejemplo completo

Consola que cuenta cuántas consultas ejecuta cada forma de carga. `Relaciones.csproj`:

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
    <PackageReference Include="Microsoft.EntityFrameworkCore.Proxies" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using System.Data.Common;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;

var contador = new ContadorDeConsultas();
var opciones = new DbContextOptionsBuilder<TareasContext>()
    .UseSqlite("Data Source=relaciones.db")
    .UseLazyLoadingProxies()
    .AddInterceptors(contador)
    .Options;

await using (var db = new TareasContext(opciones))
{
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();

    foreach (var nombre in new[] { "Hogar", "Personal", "Trabajo" })
    {
        var categoria = new Categoria { Nombre = nombre };
        categoria.Tareas.Add(new Tarea { Titulo = $"{nombre} 1" });
        categoria.Tareas.Add(new Tarea { Titulo = $"{nombre} 2" });
        db.Categorias.Add(categoria);
    }
    await db.SaveChangesAsync();
}

// 1. Lazy loading: una consulta por cada c.Tareas
await using (var db = new TareasContext(opciones))
{
    contador.Total = 0;
    var categorias = await db.Categorias.OrderBy(c => c.Nombre).ToListAsync();
    foreach (var c in categorias)
    {
        Console.WriteLine($"  {c.Nombre}: {c.Tareas.Count} tareas");
    }
    Console.WriteLine($"Lazy loading -> consultas: {contador.Total}");
}

// 2. Include: todo en una consulta con JOIN
await using (var db = new TareasContext(opciones))
{
    contador.Total = 0;
    var categorias = await db.Categorias.Include(c => c.Tareas).OrderBy(c => c.Nombre).ToListAsync();
    Console.WriteLine($"Include      -> consultas: {contador.Total}, tareas: {categorias.Sum(c => c.Tareas.Count)}");
}

// 3. AsSplitQuery: una consulta por la raíz y otra por la colección
await using (var db = new TareasContext(opciones))
{
    contador.Total = 0;
    var categorias = await db.Categorias.Include(c => c.Tareas).AsSplitQuery().OrderBy(c => c.Nombre).ToListAsync();
    Console.WriteLine($"AsSplitQuery -> consultas: {contador.Total}, tareas: {categorias.Sum(c => c.Tareas.Count)}");
}

// 4. Proyección: solo los datos pedidos, COUNT en SQL
await using (var db = new TareasContext(opciones))
{
    contador.Total = 0;
    var resumen = await db.Categorias
        .OrderBy(c => c.Nombre)
        .Select(c => new { c.Nombre, Total = c.Tareas.Count })
        .ToListAsync();
    Console.WriteLine($"Proyección   -> consultas: {contador.Total}, tareas: {resumen.Sum(r => r.Total)}");
}

public class Categoria
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public virtual List<Tarea> Tareas { get; set; } = [];
}

public class Tarea
{
    public int Id { get; set; }
    public string Titulo { get; set; } = "";
    public int CategoriaId { get; set; }
    public virtual Categoria? Categoria { get; set; }
}

public class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Categoria> Categorias => Set<Categoria>();
    public DbSet<Tarea> Tareas => Set<Tarea>();
}

public sealed class ContadorDeConsultas : DbCommandInterceptor
{
    public int Total { get; set; }

    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand command, CommandEventData eventData, InterceptionResult<DbDataReader> result)
    {
        Total++;
        return base.ReaderExecuting(command, eventData, result);
    }

    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(
        DbCommand command, CommandEventData eventData, InterceptionResult<DbDataReader> result,
        CancellationToken cancellationToken = default)
    {
        Total++;
        return base.ReaderExecutingAsync(command, eventData, result, cancellationToken);
    }
}
```

Salida:

```text
  Hogar: 2 tareas
  Personal: 2 tareas
  Trabajo: 2 tareas
Lazy loading -> consultas: 4
Include      -> consultas: 1, tareas: 6
AsSplitQuery -> consultas: 2, tareas: 6
Proyección   -> consultas: 1, tareas: 6
```

Qué observar:

* Con 3 categorías, lazy loading hizo **4** consultas: 1 + N. Con 300 categorías serían 301, y el código no cambiaría ni una línea.
* El interceptor cuenta la carga perezosa con `ReaderExecuting` (síncrono): esas consultas bloquean el hilo aunque el resto sea `async`.
* `Include` resuelve todo en 1 consulta; `AsSplitQuery`, en 2 (raíz + colección).
* La proyección también es 1 consulta, pero no materializa ninguna `Tarea`: SQL devuelve el `COUNT` ya calculado.
* Cada bloque usa un contexto nuevo. Si reutilizaras el mismo, las tareas ya rastreadas se conectarían solas a sus categorías (*fix-up*) y los conteos engañarían.

-----

## Errores comunes

**1. Leer una navegación sin cargarla.**
Qué pasa: colecciones vacías o referencias `null` que parecen datos reales.
Por qué: EF Core solo llena navegaciones con `Include`, carga explícita, lazy loading o *fix-up* de entidades ya rastreadas.
Arreglo: `Include` o, mejor para lecturas, proyección.

**2. Activar lazy loading "para que funcione".**
Qué pasa: N+1 consultas síncronas, y serializar una entidad dispara consultas en cascada.
Por qué: cada acceso a una navegación no cargada es una consulta.
Arreglo: déjalo apagado; carga explícitamente lo que necesitas. Si lo heredas, `Include` en las consultas calientes.

**3. Encadenar varios `Include` de colecciones.**
Qué pasa: miles de filas duplicadas y la advertencia de EF Core sobre consultas con varias colecciones.
Por qué: un `JOIN` con dos colecciones multiplica sus tamaños.
Arreglo: `AsSplitQuery()` o proyectar solo lo necesario.

**4. `FirstOrDefault` con `Include` y sin `OrderBy`.**
Qué pasa: el "primer" libro cambia entre ejecuciones; EF Core registra la advertencia de operador de límite sin `OrderBy`.
Por qué: SQL no garantiza orden sin `ORDER BY`.
Arreglo: filtra por clave (`FirstOrDefaultAsync(l => l.Id == id)`) u ordena antes.

**5. Devolver entidades con navegaciones bidireccionales como JSON.**
Qué pasa: `JsonException: A possible object cycle was detected...`.
Por qué: `Categoria.Tareas[0].Categoria.Tareas[0]...` no termina nunca.
Arreglo: devuelve DTOs, no entidades. `ReferenceHandler.IgnoreCycles` esconde el síntoma, no el problema.

**6. Crear la tabla intermedia a mano cuando no tiene datos propios.**
Qué pasa: más clases, más configuración y `Include(...).ThenInclude(...)` donde bastaba `Include(l => l.Autores)`.
Por qué: el tutorial era anterior a EF Core 5.
Arreglo: skip navigations; entidad explícita solo si la relación tiene atributos.

-----

## Según la versión de .NET

* **EF Core 2.1:** vuelve el lazy loading, opcional y con paquete aparte (`Proxies`).
* **EF Core 3.0:** `Include` siempre en una sola consulta (antes se dividía de forma implícita).
* **EF Core 5:** muchos a muchos con skip navigations, `Include` filtrado y `AsSplitQuery`.
* **EF Core 6:** muchos a muchos más configurable (`UsingEntity`) y mejoras de consultas divididas.
* **EF Core 7:** atributo `[DeleteBehavior]`.
* **EF Core 10:** acompaña a .NET 10; agrega los operadores `LeftJoin` y `RightJoin` para LINQ.

-----

## Cuándo sí y cuándo no

**Usa `Include` cuando:** vas a modificar la entidad y sus hijos en la misma unidad de trabajo, o necesitas el grafo completo.

**Usa proyección cuando:** el resultado va a una pantalla o una API. Es el caso más común.

**Usa `AsSplitQuery` cuando:** incluyes más de una colección o una colección muy grande.

**Usa carga explícita cuando:** decides en tiempo de ejecución si hace falta la navegación, para una entidad puntual.

**No uses lazy loading cuando:** se trata de una API, un listado o cualquier código que recorra colecciones. En la práctica, casi nunca.

-----

## Resumen en 5 líneas

1. La relación vive en la FK; las navegaciones son su reflejo en memoria.
2. EF Core no carga navegaciones si no lo pides: `Include`, carga explícita o proyección.
3. Lazy loading está apagado por defecto y produce N+1 consultas síncronas.
4. Varias colecciones en un `Include` explotan en filas: `AsSplitQuery` o proyección.
5. Muchos a muchos sin datos propios no necesita entidad intermedia desde EF Core 5.

-----

## Para profundizar

<details>
<summary>Cascada y "multiple cascade paths" en SQL Server</summary>

Si `Tarea` tiene FK obligatoria a `Categoria` y a `Usuario`, y `Categoria` también tiene FK obligatoria a `Usuario`, borrar un usuario llegaría a la tarea por dos caminos en cascada. SQL Server lo rechaza al crear la FK: *Introducing FOREIGN KEY constraint ... may cause cycles or multiple cascade paths*. La solución es elegir un camino: `OnDelete(DeleteBehavior.Restrict)` o `ClientCascade` en una de las relaciones, y decidir explícitamente qué pasa con los datos.

</details>

<details>
<summary><code>AsNoTracking</code> con <code>Include</code>: objetos repetidos</summary>

Con tracking, si diez tareas apuntan a la misma categoría, las diez comparten **una** instancia de `Categoria` (resolución de identidad). Con `AsNoTracking()`, EF Core crea una instancia por fila: diez objetos `Categoria` iguales. Si te importa, `AsNoTrackingWithIdentityResolution()` resuelve identidad sin rastrear cambios.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La relación se guarda en una FK, y en C# la recorro con navegaciones. EF Core no las carga solo: uso `Include` y `ThenInclude`, o mejor una proyección con `Select`. El lazy loading provoca el problema N+1, por eso no lo uso.

### Respuesta ampliada (semi-senior)

Modelo 1:N y 1:1 con FK explícita y navegaciones; en 1:1 configuro `HasForeignKey<T>` y EF Core crea el índice único. Para N:N uso skip navigations salvo que la relación tenga datos propios. Para leer hacia una API proyecto a DTOs, que traduce agregados a SQL y evita ciclos. Uso `Include` cuando voy a modificar el grafo, `AsSplitQuery` con varias colecciones para evitar la explosión cartesiana, y reviso el comportamiento de borrado en cascada. Lazy loading lo evito: N+1 invisible y consultas síncronas.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es el problema N+1?**
Una consulta para la lista y otra por cada elemento al acceder a una navegación; crece con los datos.

**2. ¿`Include` siempre es una sola consulta?**
Por defecto sí, salvo que uses `AsSplitQuery` o lo configures como comportamiento global.

**3. ¿Por qué `virtual` en las navegaciones?**
Para que los proxies de lazy loading puedan sobrescribir el `get`. Sin lazy loading no hace falta.

**4. ¿Cuándo crear la entidad de la tabla intermedia?**
Cuando la relación tiene datos propios, como un orden o una fecha.

-----

## Práctica

**Ejercicio 1.** Este código imprime libros con sus autores. ¿Cuántas consultas ejecuta con lazy loading activo y 100 libros? Reescríbelo con una sola consulta que traiga solo lo necesario.

```csharp
foreach (var libro in db.Libros.ToList())
{
    Console.WriteLine($"{libro.Titulo}: {string.Join(", ", libro.Autores.Select(a => a.Nombre))}");
}
```

<details>
<summary>Solución</summary>

101 consultas: una para los libros y una por cada `libro.Autores`, todas síncronas.

```csharp
var libros = await db.Libros
    .OrderBy(l => l.Titulo)
    .Select(l => new
    {
        l.Titulo,
        Autores = l.Autores.OrderBy(a => a.Nombre).Select(a => a.Nombre).ToList()
    })
    .ToListAsync(ct);

foreach (var libro in libros)
{
    Console.WriteLine($"{libro.Titulo}: {string.Join(", ", libro.Autores)}");
}
```

La proyección trae solo títulos y nombres, sin rastrear entidades, en una consulta.

</details>

**Ejercicio 2.** Modela pedidos con productos donde cada línea guarda la cantidad y el precio pagado. ¿Skip navigation o entidad intermedia?

<details>
<summary>Solución</summary>

Entidad intermedia: la relación tiene datos propios.

```csharp
public class LineaPedido
{
    public int PedidoId { get; set; }
    public Pedido? Pedido { get; set; }
    public int ProductoId { get; set; }
    public Producto? Producto { get; set; }
    public int Cantidad { get; set; }
    public decimal PrecioUnitario { get; set; }
}

modelBuilder.Entity<LineaPedido>(linea =>
{
    linea.HasKey(l => new { l.PedidoId, l.ProductoId });
    linea.Property(l => l.PrecioUnitario).HasPrecision(18, 2);
});
```

`PrecioUnitario` se copia al crear la línea: si el producto cambia de precio mañana, el pedido de hoy no debe cambiar.

</details>

-----

## Siguiente lección

[CRUD en APIs con EF Core](06-CRUD%20en%20APIs%20con%20EF%20Core.md)
