# Dapper avanzado y convivencia con EF Core

## En una frase

Dapper arma objetos relacionados a partir de un `JOIN` con *multi-mapping*, trae varios resultados en una ida con `QueryMultiple` y puede trabajar sobre la misma conexión que EF Core; la combinación típica es EF Core para escribir y Dapper para lecturas complejas, sabiendo que el change tracker **no se entera** de lo que Dapper cambia.

-----

## Antes de empezar

Conviene que ya sepas:

* Consultas y parámetros con Dapper: [Dapper: consultas y comandos](02-Dapper%20consultas%20y%20comandos.md).
* `NextResult` en ADO.NET: [ADO.NET a fondo](01-ADO%20NET%20a%20fondo.md).
* Relaciones y change tracker en EF Core: [Relaciones y carga de datos](../01-entity-framework-core/05-Relaciones%20y%20carga%20de%20datos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Multi-mapping:** mapear cada fila de un `JOIN` a varios objetos y unirlos con una función.
* **`splitOn`:** nombre de la columna donde empieza el siguiente objeto dentro de la fila.
* **`GridReader`:** lector que devuelve `QueryMultiple` para recorrer varios resultados en orden.
* **Type handler:** clase que le enseña a Dapper a convertir un tipo propio a y desde un valor de la base.
* **Modelo de lectura:** clases pensadas solo para mostrar datos (vistas, reportes), distintas de las entidades que se modifican.

-----

## El problema

Un reporte necesita tareas con su categoría. Primera versión con Dapper:

```csharp
var tareas = await conexion.QueryAsync<TareaVista>("SELECT Id, Titulo, CategoriaId FROM Tareas");
foreach (var t in tareas)
{
    t.Categoria = await conexion.QuerySingleAsync<CategoriaVista>(
        "SELECT Id, Nombre FROM Categorias WHERE Id = @Id", new { Id = t.CategoriaId });
}
```

Es el **N+1** de la lección de relaciones, escrito a mano: 1 consulta + 1 por tarea. Con 500 tareas, 501 idas a la base.

Además, el proyecto usa EF Core para escribir. Un endpoint carga una tarea con EF Core, otro servicio le cambia el título con Dapper en la misma petición, y el endpoint vuelve a consultar con EF Core:

```csharp
var tarea = await db.Tareas.SingleAsync(t => t.Id == id);           // título: "Barrer"
await conexion.ExecuteAsync("UPDATE Tareas SET Titulo = 'Barrer y trapear' WHERE Id = @id", new { id });
var otraVez = await db.Tareas.SingleAsync(t => t.Id == id);         // ¿título?
```

`otraVez.Titulo` sigue siendo **"Barrer"**. Y si después alguien modifica otra propiedad y llama a `SaveChanges`, EF Core no pisa el título (solo envía columnas modificadas), pero cualquier lógica que lea `tarea.Titulo` trabaja con un dato que ya no existe.

Las consecuencias:

* N+1 consultas porque no se sabe armar objetos relacionados desde un `JOIN`;
* datos desactualizados en memoria al mezclar herramientas sin entender el change tracker.

-----

## Cómo funciona

### 1. Multi-mapping: un `JOIN`, dos objetos por fila

```csharp
const string sql = """
    SELECT t.Id, t.Titulo, t.Completada, c.Id, c.Nombre
    FROM Tareas t
    JOIN Categorias c ON c.Id = t.CategoriaId
    ORDER BY t.Titulo
    """;

var tareas = await conexion.QueryAsync<TareaVista, CategoriaVista, TareaVista>(
    sql,
    (tarea, categoria) => { tarea.Categoria = categoria; return tarea; },
    splitOn: "Id");
```

```text
fila:   t.Id │ t.Titulo │ t.Completada ║ c.Id │ c.Nombre
        └──────── TareaVista ─────────┘ ╚══ CategoriaVista ══╝
                                         ▲
                                  splitOn: "Id" (segunda columna Id)
```

* Los genéricos son `<Primero, Segundo, Resultado>`; hay sobrecargas hasta siete objetos.
* `splitOn` indica dónde empieza cada objeto siguiente. El valor por defecto es `"Id"`, que funciona si cada tabla tiene su columna `Id` y la seleccionas **primera** en su bloque.
* Si la clave de la segunda tabla se llama distinto, indícalo: `splitOn: "CategoriaId"`. Con tres objetos: `splitOn: "CategoriaId,ResponsableId"`.

### 2. Uno a muchos: agrupar filas repetidas

Con un `JOIN` de uno a muchos, la categoría aparece una vez **por cada tarea**. Hay que reutilizar la instancia:

```csharp
var porId = new Dictionary<int, CategoriaVista>();

var categorias = (await conexion.QueryAsync<CategoriaVista, TareaVista?, CategoriaVista>(
        sql,
        (categoria, tarea) =>
        {
            if (!porId.TryGetValue(categoria.Id, out var existente))
            {
                existente = categoria;
                porId.Add(categoria.Id, existente);
            }
            if (tarea is not null)
            {
                existente.Tareas.Add(tarea);
            }
            return existente;
        },
        splitOn: "Id"))
    .Distinct()
    .ToList();
```

* El diccionario evita crear una `CategoriaVista` por fila.
* Con `LEFT JOIN`, una categoría sin tareas trae `NULL` en las columnas de la tarea: si la columna de `splitOn` es `NULL`, Dapper pasa `null` como objeto.
* `Distinct()` deja una referencia por categoría, en el orden del SQL.

Es exactamente lo que EF Core hace por ti con `Include`. En Dapper lo escribes, y tienes el mismo riesgo de explosión cartesiana si unes varias colecciones.

### 3. `QueryMultiple`: varias consultas, una ida

```csharp
using var grid = await conexion.QueryMultipleAsync("""
    SELECT COUNT(*) FROM Tareas;
    SELECT COUNT(*) FROM Tareas WHERE Completada = 0;
    SELECT Nombre FROM Categorias c WHERE NOT EXISTS (SELECT 1 FROM Tareas t WHERE t.CategoriaId = c.Id);
    """);

int total = await grid.ReadSingleAsync<int>();
int pendientes = await grid.ReadSingleAsync<int>();
var vacias = (await grid.ReadAsync<string>()).ToList();
```

Los `Read` se llaman **en el mismo orden** que las consultas. Es `NextResult` de ADO.NET con mapeo. Ideal para pantallas tipo "dashboard" que necesitan varios números.

### 4. Procedimientos almacenados

```csharp
var pendientes = await conexion.QueryAsync<TareaVista>(
    "dbo.TareasPendientes",
    new { UsuarioId = usuarioId },
    commandType: CommandType.StoredProcedure);
```

Con `CommandType.StoredProcedure`, el texto es el nombre del procedimiento y las propiedades del objeto son sus parámetros. Para parámetros de salida, usa `DynamicParameters` con `direction: ParameterDirection.Output` y léelos con `parametros.Get<int>("Total")` después de ejecutar. (SQLite no tiene procedimientos almacenados.)

### 5. Type handlers: tipos propios

```csharp
public sealed class EmailHandler : SqlMapper.TypeHandler<Email>
{
    public override void SetValue(IDbDataParameter parameter, Email? value) => parameter.Value = value?.Valor;
    public override Email Parse(object value) => new Email((string)value);
}

SqlMapper.AddTypeHandler(new EmailHandler());   // una vez, al iniciar
```

Con esto, un value object `Email` funciona como parámetro y como propiedad de resultado. Es el equivalente a `HasConversion` de EF Core. Ojo: el registro es **global** (estático) para toda la aplicación.

### 6. Dapper sobre la conexión de EF Core

```csharp
DbConnection conexion = db.Database.GetDbConnection();
var resumen = await conexion.QueryAsync<ResumenVista>(sql);
```

* Misma cadena de conexión, misma configuración, sin registrar nada nuevo en DI.
* Si EF Core tiene una transacción abierta, Dapper debe unirse a ella: `transaction: db.Database.CurrentTransaction?.GetDbTransaction()` ([lección 4](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md)).
* No hagas `Dispose` de esa conexión: le pertenece al contexto.

### 7. El change tracker no ve a Dapper

```text
EF Core (tracking)                         Base de datos
tarea "Barrer" en memoria   ── Dapper UPDATE ──►  "Barrer y trapear"
db.Tareas.Single(...)       ◄─ la fila llega ─── "Barrer y trapear"
  → devuelve la instancia YA rastreada, sin pisar sus valores: "Barrer"
```

Una consulta con tracking que encuentra una entidad ya rastreada devuelve **esa** instancia y no sobrescribe sus propiedades con los datos nuevos. Opciones:

* `await db.Entry(tarea).ReloadAsync()` para refrescar una entidad;
* `AsNoTracking()` para leer el valor actual;
* mejor aún: no mezclar escrituras con Dapper y entidades rastreadas en la misma unidad de trabajo.

### 8. Una división de trabajo razonable

| Operación | Herramienta | Por qué |
| --- | --- | --- |
| Crear, editar, borrar entidades con reglas | EF Core | Tracking, relaciones, concurrencia optimista, migraciones |
| Listados simples | EF Core con proyección | Tipado, sin SQL a mano |
| Reportes, CTE, funciones de ventana, SQL específico del motor | Dapper | SQL exacto y legible |
| Cambios masivos simples | `ExecuteUpdate`/`ExecuteDelete` de EF Core | Una sentencia, sin cargar entidades |

Encaja natural con [CQRS](../../02-architecture-and-design-patterns/06-cqrs-y-mediator/02-CQRS%20y%20mediator%20con%20MediatR.md): los command handlers usan EF Core; algunos query handlers, Dapper. Las migraciones siguen siendo de EF Core: Dapper no las necesita, solo lee el esquema que EF Core creó.

-----

## Ejemplo completo

Consola donde EF Core crea el esquema y los datos, y Dapper lee sobre la misma conexión. `DapperEf.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Dapper" Version="2.1.35" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using Dapper;
using Microsoft.EntityFrameworkCore;

var opciones = new DbContextOptionsBuilder<TareasContext>().UseSqlite("Data Source=dapper-ef.db").Options;

// EF Core: esquema y datos (lado de escritura)
await using (var db = new TareasContext(opciones))
{
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();

    var hogar = new Categoria { Nombre = "Hogar" };
    hogar.Tareas.Add(new Tarea { Titulo = "Pagar la luz", Completada = true });
    hogar.Tareas.Add(new Tarea { Titulo = "Barrer" });
    var trabajo = new Categoria { Nombre = "Trabajo" };
    trabajo.Tareas.Add(new Tarea { Titulo = "Revisar PR" });
    db.Categorias.AddRange(hogar, trabajo, new Categoria { Nombre = "Personal" });
    await db.SaveChangesAsync();
}

await using (var db = new TareasContext(opciones))
{
    var conexion = db.Database.GetDbConnection();   // la conexión de EF Core, para Dapper

    // 1. Multi-mapping 1:1
    var tareas = await conexion.QueryAsync<TareaVista, CategoriaVista, TareaVista>(
        """
        SELECT t.Id, t.Titulo, t.Completada, c.Id, c.Nombre
        FROM Tareas t
        JOIN Categorias c ON c.Id = t.CategoriaId
        ORDER BY t.Titulo
        """,
        (tarea, categoria) => { tarea.Categoria = categoria; return tarea; },
        splitOn: "Id");

    Console.WriteLine("Tareas con su categoría:");
    foreach (var t in tareas)
    {
        Console.WriteLine($"  {t.Titulo} [{t.Categoria!.Nombre}]");
    }

    // 2. Multi-mapping 1:N con LEFT JOIN y agrupación
    var porId = new Dictionary<int, CategoriaVista>();
    var categorias = (await conexion.QueryAsync<CategoriaVista, TareaVista?, CategoriaVista>(
            """
            SELECT c.Id, c.Nombre, t.Id, t.Titulo, t.Completada
            FROM Categorias c
            LEFT JOIN Tareas t ON t.CategoriaId = c.Id
            ORDER BY c.Nombre, t.Titulo
            """,
            (categoria, tarea) =>
            {
                if (!porId.TryGetValue(categoria.Id, out var existente))
                {
                    existente = categoria;
                    porId.Add(categoria.Id, existente);
                }
                if (tarea is not null)
                {
                    existente.Tareas.Add(tarea);
                }
                return existente;
            },
            splitOn: "Id"))
        .Distinct()
        .ToList();

    Console.WriteLine("Categorías con sus tareas:");
    foreach (var c in categorias)
    {
        string lista = c.Tareas.Count == 0 ? "(sin tareas)" : string.Join(", ", c.Tareas.Select(t => t.Titulo));
        Console.WriteLine($"  {c.Nombre}: {lista}");
    }

    // 3. QueryMultiple: tres consultas en una ida
    using (var grid = await conexion.QueryMultipleAsync("""
        SELECT COUNT(*) FROM Tareas;
        SELECT COUNT(*) FROM Tareas WHERE Completada = 0;
        SELECT Nombre FROM Categorias c WHERE NOT EXISTS (SELECT 1 FROM Tareas t WHERE t.CategoriaId = c.Id);
        """))
    {
        int total = await grid.ReadSingleAsync<int>();
        int pendientes = await grid.ReadSingleAsync<int>();
        var vacias = await grid.ReadAsync<string>();
        Console.WriteLine($"Resumen -> total: {total}, pendientes: {pendientes}, sin tareas: {string.Join(", ", vacias)}");
    }

    // 4. El change tracker no ve lo que cambia Dapper
    var barrer = await db.Tareas.SingleAsync(t => t.Titulo == "Barrer");
    await conexion.ExecuteAsync(
        "UPDATE Tareas SET Titulo = @Nuevo WHERE Id = @Id", new { Nuevo = "Barrer y trapear", barrer.Id });

    var otraVez = await db.Tareas.SingleAsync(t => t.Id == barrer.Id);
    Console.WriteLine($"EF (rastreada): {otraVez.Titulo}");
    Console.WriteLine($"Base de datos:  {await conexion.QuerySingleAsync<string>("SELECT Titulo FROM Tareas WHERE Id = @Id", new { barrer.Id })}");

    await db.Entry(barrer).ReloadAsync();
    Console.WriteLine($"Tras Reload:    {barrer.Titulo}");
}

// ---------- Entidades (EF Core) ----------
public class Categoria
{
    public int Id { get; set; }
    public required string Nombre { get; set; }
    public List<Tarea> Tareas { get; } = [];
}

public class Tarea
{
    public int Id { get; set; }
    public required string Titulo { get; set; }
    public bool Completada { get; set; }
    public int CategoriaId { get; set; }
    public Categoria? Categoria { get; set; }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Categoria> Categorias => Set<Categoria>();
    public DbSet<Tarea> Tareas => Set<Tarea>();
}

// ---------- Modelos de lectura (Dapper) ----------
public sealed class CategoriaVista
{
    public int Id { get; init; }
    public string Nombre { get; init; } = "";
    public List<TareaVista> Tareas { get; } = [];
}

public sealed class TareaVista
{
    public int Id { get; init; }
    public string Titulo { get; init; } = "";
    public bool Completada { get; init; }
    public CategoriaVista? Categoria { get; set; }
}
```

Salida:

```text
Tareas con su categoría:
  Barrer [Hogar]
  Pagar la luz [Hogar]
  Revisar PR [Trabajo]
Categorías con sus tareas:
  Hogar: Barrer, Pagar la luz
  Personal: (sin tareas)
  Trabajo: Revisar PR
Resumen -> total: 3, pendientes: 2, sin tareas: Personal
EF (rastreada): Barrer
Base de datos:  Barrer y trapear
Tras Reload:    Barrer y trapear
```

Qué observar:

* Tres tareas con su categoría salieron de **una** consulta, no de 1 + 3.
* "Hogar" aparece una sola vez con dos tareas gracias al diccionario; "Personal" aparece sin tareas gracias al `LEFT JOIN` y al `null` de Dapper.
* `QueryMultiple` trajo tres resultados distintos en una ida.
* La segunda consulta de EF Core devolvió la instancia rastreada con el título viejo; la base ya tenía el nuevo. `ReloadAsync` los volvió a alinear.
* Las entidades (`Tarea`, `Categoria`) y los modelos de lectura (`TareaVista`, `CategoriaVista`) son clases distintas: cada lado tiene la forma que necesita.

-----

## Errores comunes

**1. Armar relaciones con una consulta por elemento.**
Qué pasa: N+1 consultas, igual que el lazy loading.
Por qué: se consulta la relación dentro de un bucle.
Arreglo: `JOIN` + multi-mapping, o dos consultas con `QueryMultiple` y unión en memoria por clave.

**2. `splitOn` incorrecto.**
Qué pasa: `ArgumentException: When using the multi-mapping APIs ensure you set the splitOn param if you have keys other than Id`, u objetos con propiedades vacías.
Por qué: Dapper no encuentra la columna donde cortar, o corta en la equivocada.
Arreglo: selecciona la clave primero en cada bloque e indica `splitOn` con su nombre.

**3. Olvidar agrupar en uno a muchos.**
Qué pasa: una categoría repetida por cada tarea, cada una con una sola tarea en su lista.
Por qué: Dapper crea objetos nuevos por fila; no hace resolución de identidad.
Arreglo: diccionario por clave y `Distinct()`.

**4. Leer el `GridReader` en otro orden.**
Qué pasa: `InvalidCastException` o datos de la consulta equivocada.
Por qué: los resultados se consumen secuencialmente.
Arreglo: un `Read` por consulta, en el orden del SQL.

**5. Escribir con Dapper y seguir usando entidades rastreadas.**
Qué pasa: EF Core muestra valores viejos; reglas que deciden con datos que ya cambiaron.
Por qué: el change tracker no sabe que la fila cambió y no sobrescribe instancias rastreadas.
Arreglo: no mezclar en la misma unidad de trabajo; si es inevitable, `ReloadAsync` o `ChangeTracker.Clear()`.

**6. Hacer `Dispose` de la conexión de EF Core.**
Qué pasa: el contexto falla en la siguiente operación.
Por qué: `GetDbConnection()` devuelve la conexión que el contexto administra.
Arreglo: úsala sin `using`; el contexto la libera al terminar.

-----

## Según la versión de .NET

* **Dapper 1.x:** multi-mapping, `QueryMultiple` y type handlers desde las primeras versiones.
* **EF Core 1.0:** `Database.GetDbConnection()` y `Database.UseTransaction(...)` para compartir conexión y transacción con ADO.NET.
* **EF Core 7:** `SqlQuery<T>` para tipos escalares.
* **EF Core 8:** `SqlQuery<T>` con tipos no mapeados: para consultas planas con SQL propio ya no hace falta Dapper.
* **Dapper 2.1 (2023):** anotaciones de nulabilidad y mejoras de rendimiento.

-----

## Cuándo sí y cuándo no

**Usa multi-mapping cuando:** necesitas objetos relacionados desde SQL propio y el `JOIN` no multiplica demasiado las filas.

**Usa `QueryMultiple` cuando:** una pantalla necesita varios resultados independientes.

**Combina Dapper con EF Core cuando:** EF Core resuelve bien la escritura y algunas lecturas necesitan SQL específico. Comparte la conexión y respeta la regla de no mezclar escrituras con entidades rastreadas.

**No agregues Dapper cuando:** EF Core con proyección o `SqlQuery<T>` ya resuelve la consulta. Dos herramientas son dos formas de equivocarse.

-----

## Resumen en 5 líneas

1. Multi-mapping convierte cada fila de un `JOIN` en varios objetos; `splitOn` marca dónde empieza cada uno.
2. En uno a muchos, agrupa con un diccionario: Dapper no resuelve identidad.
3. `QueryMultiple` trae varios resultados en una ida; se leen en orden.
4. Dapper puede usar `db.Database.GetDbConnection()` sin crear otra conexión.
5. El change tracker no ve los cambios de Dapper: no los mezcles en la misma unidad de trabajo.

-----

## Para profundizar

<details>
<summary>Uno a muchos sin <code>JOIN</code>: dos consultas y unión en memoria</summary>

Con colecciones grandes, el `JOIN` repite las columnas del padre en cada fila. Alternativa: `QueryMultiple` con `SELECT ... FROM Categorias` y `SELECT ... FROM Tareas WHERE CategoriaId IN (...)`, y luego `tareas.ToLookup(t => t.CategoriaId)` para asignar cada grupo a su categoría. Son dos consultas sin duplicados: la misma idea que `AsSplitQuery` en EF Core.

</details>

<details>
<summary>¿Cuándo <code>SqlQuery&lt;T&gt;</code> de EF Core alcanza?</summary>

`db.Database.SqlQuery<ResumenVista>($"SELECT ...")` (EF Core 8+) mapea SQL propio a un tipo no registrado en el modelo, con parámetros seguros por interpolación. No soporta multi-mapping ni varios resultados, pero para una consulta plana evita agregar otra dependencia. Si toda tu necesidad de Dapper son dos reportes planos, probablemente no necesitas Dapper.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Con multi-mapping, Dapper convierte cada fila de un `JOIN` en varios objetos y los uno con una función; `splitOn` indica dónde empieza cada uno. `QueryMultiple` trae varios resultados en una sola consulta. Se puede usar Dapper con la conexión de EF Core.

### Respuesta ampliada (semi-senior)

Para relaciones uso multi-mapping con `splitOn` explícito, y en uno a muchos agrupo con un diccionario porque Dapper no resuelve identidad; si el `JOIN` multiplica filas, hago dos consultas con `QueryMultiple` y uno en memoria. En proyectos con EF Core, comparto la conexión con `GetDbConnection()` y, si hay transacción, paso `CurrentTransaction.GetDbTransaction()`. Divido el trabajo: EF Core para escribir y lecturas simples, Dapper para reportes con SQL específico. Sé que el change tracker no ve los cambios de Dapper, así que no mezclo ambas escrituras en una unidad de trabajo.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es `splitOn`?**
La columna donde Dapper empieza a llenar el siguiente objeto de la fila.

**2. ¿Dapper resuelve identidad como EF Core?**
No: crea un objeto por fila. La deduplicación la haces tú.

**3. ¿Puedo usar Dapper dentro de una transacción de EF Core?**
Sí, pasando `db.Database.CurrentTransaction.GetDbTransaction()` como `transaction`.

**4. ¿Por qué EF Core devolvió el valor viejo después de un `UPDATE` con Dapper?**
Porque la entidad ya estaba rastreada y una consulta con tracking no sobrescribe instancias existentes.

-----

## Práctica

**Ejercicio 1.** Agrega a `TareaVista` un responsable (`ResponsableVista` con `Id` y `Nombre`) y escribe el multi-mapping de tres objetos. La tabla `Responsables` tiene la clave `Id` y `Tareas` la FK `ResponsableId`.

<details>
<summary>Solución</summary>

```csharp
var tareas = await conexion.QueryAsync<TareaVista, CategoriaVista, ResponsableVista, TareaVista>(
    """
    SELECT t.Id, t.Titulo, t.Completada, c.Id, c.Nombre, r.Id, r.Nombre
    FROM Tareas t
    JOIN Categorias c ON c.Id = t.CategoriaId
    JOIN Responsables r ON r.Id = t.ResponsableId
    ORDER BY t.Titulo
    """,
    (tarea, categoria, responsable) =>
    {
        tarea.Categoria = categoria;
        tarea.Responsable = responsable;
        return tarea;
    },
    splitOn: "Id,Id");
```

Con `splitOn: "Id,Id"`, Dapper corta en la segunda y la tercera columna `Id`. Si las claves tuvieran otros nombres, irían en orden: `splitOn: "CategoriaId,ResponsableId"`.

</details>

**Ejercicio 2.** Un endpoint hace esto en la misma petición. ¿Qué guarda la base al final?

```csharp
var tarea = await db.Tareas.SingleAsync(t => t.Id == 1);       // Titulo = "A", Completada = false
await conexion.ExecuteAsync("UPDATE Tareas SET Titulo = 'B' WHERE Id = 1");
tarea.Completada = true;
await db.SaveChangesAsync();
```

<details>
<summary>Solución</summary>

La fila queda con `Titulo = 'B'` y `Completada = 1`. EF Core envía un `UPDATE` solo de `Completada`, la única propiedad marcada como modificada, así que no pisa el título. Pero en memoria `tarea.Titulo` sigue siendo `"A"`: cualquier respuesta construida con esa entidad muestra un dato viejo, y si alguna regla dependía del título, decidió con información incorrecta. Si en lugar de modificar una propiedad hubieras usado `db.Update(tarea)`, EF Core habría enviado **todas** las columnas y el título volvería a ser `'A'`.

</details>

-----

## Siguiente lección

[Transacciones con ADO.NET, Dapper y EF Core](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md)
