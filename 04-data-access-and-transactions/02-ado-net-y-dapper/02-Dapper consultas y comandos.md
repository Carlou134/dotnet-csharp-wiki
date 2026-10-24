# Dapper: consultas y comandos

## En una frase

Dapper es un micro-ORM: agrega métodos de extensión a `DbConnection` que ejecutan **tu** SQL, convierten un objeto anónimo en parámetros y cada fila en un objeto; te ahorra el código repetitivo de ADO.NET sin esconder el SQL ni rastrear cambios.

-----

## Antes de empezar

Conviene que ya sepas:

* Conexiones, comandos, readers y `DBNull`: [ADO.NET a fondo](01-ADO%20NET%20a%20fondo.md).
* Qué es un micro-ORM frente a un ORM: [ADO.NET, ORMs y EF Core](../01-entity-framework-core/01-ADO%20NET%20ORMs%20y%20EF%20Core.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Dapper:** biblioteca de Stack Overflow que mapea resultados de SQL a objetos con métodos de extensión sobre `DbConnection`.
* **Materialización:** convertir una fila del resultado en un objeto de C#.
* **Objeto de parámetros:** objeto (normalmente anónimo) cuyas propiedades Dapper convierte en parámetros SQL con el mismo nombre.
* **`DynamicParameters`:** colección de parámetros que se arma en tiempo de ejecución, útil para filtros opcionales.
* **Consulta con *buffer*:** Dapper lee todas las filas a una lista antes de devolverla (comportamiento por defecto).

-----

## El problema

El mismo listado de tareas, en ADO.NET, ocupa esto:

```csharp
await using var cmd = conexion.CreateCommand();
cmd.CommandText = "SELECT id, titulo, descripcion, completada FROM tareas WHERE categoria_id = $cat ORDER BY id";
cmd.Parameters.Add("$cat", SqliteType.Integer).Value = categoriaId;

await using var reader = await cmd.ExecuteReaderAsync(ct);
int colId = reader.GetOrdinal("id");
int colTitulo = reader.GetOrdinal("titulo");
int colDescripcion = reader.GetOrdinal("descripcion");
int colCompletada = reader.GetOrdinal("completada");

var lista = new List<Tarea>();
while (await reader.ReadAsync(ct))
{
    lista.Add(new Tarea
    {
        Id = reader.GetInt32(colId),
        Titulo = reader.GetString(colTitulo),
        Descripcion = reader.IsDBNull(colDescripcion) ? null : reader.GetString(colDescripcion),
        Completada = reader.GetBoolean(colCompletada)
    });
}
```

Diecinueve líneas, de las cuales **una** es la consulta. Multiplícalo por cincuenta consultas:

* cada copia es una oportunidad de olvidar un `IsDBNull` o equivocar un ordinal;
* agregar una columna obliga a tocar el SQL **y** el bucle de mapeo;
* la intención (qué datos pides) queda enterrada en código de plomería.

Con Dapper:

```csharp
var lista = await conexion.QueryAsync<Tarea>(
    "SELECT id, titulo, descripcion, completada FROM tareas WHERE categoria_id = @CategoriaId ORDER BY id",
    new { CategoriaId = categoriaId });
```

-----

## Cómo funciona

### 1. Qué hace Dapper por dentro

```text
QueryAsync<Tarea>(sql, new { CategoriaId = 1 })
   │
   ├─► crea DbCommand con el SQL
   ├─► por cada propiedad del objeto: parámetro @CategoriaId = 1 (null → DBNull.Value)
   ├─► abre la conexión si estaba cerrada
   ├─► ExecuteReader
   ├─► por cada fila: new Tarea + asigna columnas a propiedades por NOMBRE
   │      (código IL generado una vez por consulta y tipo, y cacheado)
   └─► cierra la conexión si la abrió él
```

No hay `DbContext`, ni change tracker, ni traducción de LINQ. Es ADO.NET con el mapeo automatizado.

### 2. Los métodos principales

| Método | Devuelve | Si no hay filas | Si hay varias |
| --- | --- | --- | --- |
| `QueryAsync<T>` | `IEnumerable<T>` | Vacío | Todas |
| `QueryFirstOrDefaultAsync<T>` | `T?` | `default` | La primera |
| `QuerySingleOrDefaultAsync<T>` | `T?` | `default` | `InvalidOperationException` |
| `QueryFirstAsync<T>` / `QuerySingleAsync<T>` | `T` | `InvalidOperationException` | Single: excepción |
| `ExecuteAsync` | `int`: filas afectadas | — | — |
| `ExecuteScalarAsync<T>` | `T`: primera columna de la primera fila | `default` | La primera |

Mismo criterio que en EF Core: `Single` cuando "debe haber como mucho una", `First` cuando tomas la primera de varias **ordenadas**.

### 3. Parámetros: un objeto anónimo

```csharp
await conexion.ExecuteAsync(
    "UPDATE tareas SET completada = 1 WHERE categoria_id = @CategoriaId AND completada = @Completada",
    new { CategoriaId = 2, Completada = false });
```

* Cada propiedad se convierte en un parámetro con el mismo nombre. No hay concatenación: **es seguro contra inyección SQL**.
* `null` se envía como `NULL`: Dapper hace el `DBNull.Value` por ti.
* Con una **lista** en una cláusula `IN`, Dapper expande los parámetros:

```csharp
var titulos = await conexion.QueryAsync<string>(
    "SELECT titulo FROM tareas WHERE id IN @Ids", new { Ids = new[] { 1, 3 } });
// se ejecuta: ... WHERE id IN (@Ids1, @Ids2)
```

Escribe `IN @Ids`, **sin paréntesis**: Dapper los agrega.

### 4. Ejecutar para cada elemento de una lista

```csharp
int insertadas = await conexion.ExecuteAsync(
    "INSERT INTO tareas (titulo, categoria_id) VALUES (@Titulo, @CategoriaId)",
    nuevas);   // un IEnumerable de objetos
```

Dapper ejecuta el comando **una vez por elemento** y suma las filas afectadas. Es cómodo, pero no es un *bulk insert*: con 10 000 elementos son 10 000 sentencias. Envuélvelo en una transacción ([lección 4](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md)) o usa la herramienta de carga masiva del motor (`SqlBulkCopy`, `COPY` en PostgreSQL).

### 5. Mapeo por nombre

Dapper asigna cada columna a la propiedad (o parámetro de constructor) con el mismo nombre, sin distinguir mayúsculas. Si la base usa `snake_case`:

| Opción | Código |
| --- | --- |
| Alias en el SQL | `SELECT categoria_id AS CategoriaId ...` |
| Configuración global | `DefaultTypeMap.MatchNamesWithUnderscores = true;` (una vez al iniciar) |

Una columna sin propiedad correspondiente se ignora; una propiedad sin columna queda con su valor por defecto. **No hay error en ninguno de los dos casos**: un alias mal escrito produce una propiedad vacía en silencio.

### 6. Records y tipos del proveedor

Con clases de propiedades `{ get; set; }` o `{ get; init; }`, Dapper convierte tipos compatibles (un `long` de SQLite a una propiedad `int` o `bool`). Con un **record posicional**, usa el constructor, y los tipos de los parámetros deben coincidir **exactamente** con los de las columnas:

```csharp
public sealed record TareaDto(int Id, string Titulo);

await conexion.QueryAsync<TareaDto>("SELECT id, titulo FROM tareas");   // SQLite
// InvalidOperationException: A parameterless default constructor or one matching signature
// (System.Int64 id, System.String titulo) is required for TareaDto materialization
```

SQLite devuelve todos los enteros como `Int64`. Soluciones: usar `long` en el record, una clase con propiedades o un `CAST` en el SQL. Con SQL Server, un `INT` llega como `Int32` y el record funciona.

### 7. Filtros opcionales con `DynamicParameters`

```csharp
var sql = new StringBuilder("SELECT COUNT(*) FROM tareas WHERE 1 = 1");
var parametros = new DynamicParameters();

if (soloPendientes)
{
    sql.Append(" AND completada = 0");
}
if (categoriaId is not null)
{
    sql.Append(" AND categoria_id = @CategoriaId");
    parametros.Add("CategoriaId", categoriaId);
}

int total = await conexion.ExecuteScalarAsync<int>(sql.ToString(), parametros);
```

Construyes **la estructura** del SQL con texto fijo escrito por ti; los **valores** siempre van como parámetros. Concatenar un valor que vino del usuario sigue siendo inyección SQL, uses Dapper o no.

### 8. Cancelación

Las sobrecargas simples no reciben `CancellationToken`. Para pasarlo, usa `CommandDefinition`:

```csharp
var tareas = await conexion.QueryAsync<Tarea>(
    new CommandDefinition(sql, new { CategoriaId = 1 }, cancellationToken: ct));
```

### 9. Dapper frente a EF Core

| | Dapper | EF Core |
| --- | --- | --- |
| Quién escribe el SQL | Tú | EF Core (desde LINQ) |
| Rastreo de cambios | No | Sí |
| `INSERT`/`UPDATE` | Los escribes tú | `SaveChanges` los genera |
| Migraciones | No (usa otra herramienta: DbUp, FluentMigrator, scripts) | Sí |
| Errores de nombres de columna | En ejecución (o propiedad vacía) | El modelo está tipado |
| Control del SQL exacto | Total | Parcial (`FromSql`, `ExecuteSql`) |
| Sobrecarga sobre ADO.NET | Mínima | Mayor (traducción, materialización, tracking) |

No compiten: Dapper brilla en lecturas complejas o reportes con SQL afinado; EF Core, en el modelo de escritura con relaciones y migraciones. Muchos proyectos usan los dos ([lección 3](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md)).

-----

## Ejemplo completo

Consola con Dapper sobre SQLite. `DapperBasico.csproj`:

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
    <PackageReference Include="Microsoft.Data.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using System.Text;
using Dapper;
using Microsoft.Data.Sqlite;

DefaultTypeMap.MatchNamesWithUnderscores = true;   // categoria_id → CategoriaId

// Conexión cerrada: Dapper la abre y la cierra en cada llamada
await using var conexion = new SqliteConnection("Data Source=dapper.db");

await conexion.ExecuteAsync("""
    DROP TABLE IF EXISTS tareas;
    CREATE TABLE tareas (
        id           INTEGER PRIMARY KEY,
        titulo       TEXT    NOT NULL,
        descripcion  TEXT    NULL,
        completada   INTEGER NOT NULL DEFAULT 0,
        categoria_id INTEGER NOT NULL
    );
    """);

// 1. Execute con una lista: una sentencia por elemento
var nuevas = new[]
{
    new { Titulo = "Estudiar Dapper", Descripcion = (string?)null, CategoriaId = 1 },
    new { Titulo = "Pagar la luz", Descripcion = (string?)"Vence el 10", CategoriaId = 2 },
    new { Titulo = "Revisar PR", Descripcion = (string?)"Antes del viernes", CategoriaId = 1 }
};
int insertadas = await conexion.ExecuteAsync(
    "INSERT INTO tareas (titulo, descripcion, categoria_id) VALUES (@Titulo, @Descripcion, @CategoriaId)",
    nuevas);
Console.WriteLine($"Insertadas: {insertadas}");

// 2. ExecuteScalar con RETURNING: el id generado
int id = await conexion.ExecuteScalarAsync<int>(
    "INSERT INTO tareas (titulo, categoria_id) VALUES (@Titulo, @CategoriaId) RETURNING id",
    new { Titulo = "Comprar pan", CategoriaId = 2 });
Console.WriteLine($"Id generado: {id}");

// 3. Query<T>: filas → objetos por nombre de columna
var pendientes = await conexion.QueryAsync<Tarea>(
    "SELECT id, titulo, descripcion, completada, categoria_id FROM tareas WHERE completada = @Completada ORDER BY id",
    new { Completada = false });
Console.WriteLine("Pendientes:");
foreach (var t in pendientes)
{
    Console.WriteLine($"  {t.Id}. {t.Titulo} - {t.Descripcion ?? "(sin descripción)"}");
}

// 4. QuerySingleOrDefault: null si no existe
var noExiste = await conexion.QuerySingleOrDefaultAsync<Tarea>(
    "SELECT id, titulo, descripcion, completada, categoria_id FROM tareas WHERE id = @Id", new { Id = 99 });
Console.WriteLine($"Id 99: {(noExiste is null ? "no existe" : noExiste.Titulo)}");

// 5. IN con una lista: Dapper expande los parámetros
var titulos = await conexion.QueryAsync<string>(
    "SELECT titulo FROM tareas WHERE id IN @Ids ORDER BY id", new { Ids = new[] { 1, 3 } });
Console.WriteLine($"Ids 1 y 3: {string.Join(", ", titulos)}");

// 6. Execute devuelve filas afectadas
int completadas = await conexion.ExecuteAsync(
    "UPDATE tareas SET completada = 1 WHERE categoria_id = @CategoriaId", new { CategoriaId = 2 });
Console.WriteLine($"Completadas de la categoría 2: {completadas}");

// 7. Filtros opcionales: estructura fija, valores como parámetros
Console.WriteLine($"Pendientes de la categoría 1: {await ContarAsync(conexion, soloPendientes: true, categoriaId: 1)}");

static async Task<int> ContarAsync(SqliteConnection conexion, bool soloPendientes, int? categoriaId)
{
    var sql = new StringBuilder("SELECT COUNT(*) FROM tareas WHERE 1 = 1");
    var parametros = new DynamicParameters();
    if (soloPendientes)
    {
        sql.Append(" AND completada = 0");
    }
    if (categoriaId is not null)
    {
        sql.Append(" AND categoria_id = @CategoriaId");
        parametros.Add("CategoriaId", categoriaId);
    }
    return await conexion.ExecuteScalarAsync<int>(sql.ToString(), parametros);
}

public sealed class Tarea
{
    public int Id { get; init; }
    public string Titulo { get; init; } = "";
    public string? Descripcion { get; init; }
    public bool Completada { get; init; }
    public int CategoriaId { get; init; }
}
```

Salida:

```text
Insertadas: 3
Id generado: 4
Pendientes:
  1. Estudiar Dapper - (sin descripción)
  2. Pagar la luz - Vence el 10
  3. Revisar PR - Antes del viernes
  4. Comprar pan - (sin descripción)
Id 99: no existe
Ids 1 y 3: Estudiar Dapper, Revisar PR
Completadas de la categoría 2: 2
Pendientes de la categoría 1: 2
```

Qué observar:

* El mapeo de `descripcion` nula, `completada` (un `INTEGER` de SQLite) a `bool` y `categoria_id` a `CategoriaId` no necesitó ni una línea de código.
* `ExecuteAsync` con un arreglo hizo tres `INSERT` y sumó las filas.
* `IN @Ids` funcionó sin paréntesis porque Dapper expandió la lista.
* La conexión nunca se abrió a mano: Dapper la abre y la cierra en cada llamada si la encuentra cerrada. Si haces muchas llamadas seguidas, abrirla tú una vez evita repetir ese trabajo (con pool, es barato igual).
* `Tarea` es una clase con `init`, no un record posicional: con SQLite, el record exigiría `long` en lugar de `int`.

-----

## Errores comunes

**1. Concatenar valores en el SQL "porque Dapper ya es seguro".**
Qué pasa: inyección SQL.
Por qué: Dapper solo protege lo que viaja en el objeto de parámetros; el texto lo escribiste tú.
Arreglo: valores siempre como `@Parametro`; concatena solo estructura fija que tú controlas.

**2. Escribir `IN (@Ids)` con paréntesis.**
Qué pasa: error de sintaxis SQL o una consulta que no encuentra nada, según el motor.
Por qué: Dapper ya agrega los paréntesis al expandir la lista.
Arreglo: `WHERE id IN @Ids`.

**3. Records posicionales con tipos que no coinciden.**
Qué pasa: `InvalidOperationException: A parameterless default constructor or one matching signature (...) is required`.
Por qué: con constructor, Dapper exige los tipos exactos de las columnas (`Int64` en SQLite).
Arreglo: tipos que coincidan con el proveedor, clase con propiedades o `CAST` en el SQL.

**4. Alias o nombres de columna mal escritos.**
Qué pasa: propiedades en `null` o `0`, sin ningún error.
Por qué: Dapper ignora columnas sin propiedad y deja propiedades sin columna en su valor por defecto.
Arreglo: alias explícitos, `MatchNamesWithUnderscores` y pruebas de integración contra la base real.

**5. Creer que `ExecuteAsync(sql, lista)` es una inserción masiva.**
Qué pasa: miles de sentencias, lentas, y sin transacción cada una se confirma por separado.
Por qué: Dapper itera la lista y ejecuta el comando por elemento.
Arreglo: transacción alrededor, o `SqlBulkCopy` / `COPY` para volúmenes grandes.

**6. `SELECT *` hacia un DTO.**
Qué pasa: viajan columnas que no se usan, y un cambio de esquema cambia silenciosamente lo que se llena.
Por qué: Dapper mapea lo que llega.
Arreglo: lista explícita de columnas.

-----

## Según la versión de .NET

* **Dapper 1.0 (2011):** creado por Stack Overflow para sus páginas de mayor tráfico.
* **Dapper 1.50 (2016):** soporte para .NET Core.
* **Dapper 2.0 (2019):** limpieza de API y de dependencias.
* **Dapper 2.0.138 / 2.1 (2023):** `QueryUnbufferedAsync` para leer como `IAsyncEnumerable<T>`, anotaciones de nulabilidad y mejoras de rendimiento.
* **Dapper.AOT:** generador de código (en evolución) que reemplaza el IL dinámico, para escenarios Native AOT.
* **C# 11:** literales `"""` para SQL de varias líneas.

-----

## Cuándo sí y cuándo no

**Usa Dapper cuando:** escribes SQL a mano (reportes, consultas con funciones de ventana, CTE), el esquema no lo controlas tú, o mediste que una consulta de EF Core necesita SQL específico.

**No uses Dapper cuando:** el trabajo es CRUD con relaciones y quieres migraciones, rastreo de cambios y consultas tipadas: ahí escribirías a mano lo que EF Core ya genera.

**No lo elijas "porque es más rápido" sin medir:** la diferencia con EF Core suele ser pequeña frente al costo de la consulta en la base.

-----

## Resumen en 5 líneas

1. Dapper ejecuta tu SQL y mapea filas a objetos por nombre de columna.
2. Los valores viajan en un objeto de parámetros: seguro contra inyección y con `NULL` automático.
3. `Query*` para leer, `Execute` para escribir, `ExecuteScalar` para un valor.
4. `IN @Ids` expande listas; `DynamicParameters` arma filtros opcionales.
5. Sin tracking ni migraciones: escribes todo el SQL, incluidos `INSERT` y `UPDATE`.

-----

## Para profundizar

<details>
<summary><code>buffered</code> y <code>QueryUnbufferedAsync</code></summary>

Por defecto, `QueryAsync<T>` lee **todas** las filas a una lista antes de devolverlas y libera la conexión. Para millones de filas, `QueryUnbufferedAsync<T>` devuelve un `IAsyncEnumerable<T>`: procesas fila por fila con `await foreach` y la memoria se mantiene constante. El costo es que la conexión queda ocupada (con el reader abierto) mientras recorres.

</details>

<details>
<summary>¿Por qué Dapper es rápido?</summary>

La primera vez que ejecuta una combinación de SQL + tipo, genera con `System.Reflection.Emit` un método que lee las columnas y asigna las propiedades, y lo guarda en caché. Las ejecuciones siguientes llaman a ese método compilado: casi el mismo código que escribirías a mano con ADO.NET. Por eso es incompatible con Native AOT, donde no se puede generar código en tiempo de ejecución; Dapper.AOT resuelve eso generando el código al compilar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Dapper es un micro-ORM: escribo el SQL y Dapper convierte los resultados en objetos. Uso `QueryAsync<T>` para leer, `ExecuteAsync` para escribir y paso los valores en un objeto anónimo, así quedan como parámetros y no hay inyección SQL.

### Respuesta ampliada (semi-senior)

Dapper extiende `DbConnection`; genera y cachea IL para mapear columnas a propiedades por nombre, así que su costo sobre ADO.NET es mínimo. Los valores van como parámetros (también listas con `IN @Ids` y `DynamicParameters` para filtros opcionales), uso `CommandDefinition` para pasar el `CancellationToken` y `MatchNamesWithUnderscores` con esquemas `snake_case`. Conozco sus trampas: errores silenciosos por alias mal escritos, records posicionales que exigen tipos exactos y `Execute` con listas que no es un *bulk insert*. Lo uso para lecturas con SQL afinado y dejo la escritura con relaciones a EF Core.

### Preguntas frecuentes de seguimiento

**1. ¿Dapper rastrea cambios?**
No. Para guardar, escribes el `UPDATE`.

**2. ¿Hay que abrir la conexión antes?**
No es obligatorio: si está cerrada, Dapper la abre y la cierra. Si la abres tú, Dapper la deja abierta.

**3. ¿Cómo se pasa una lista a un `IN`?**
`WHERE id IN @Ids` con `new { Ids = lista }`, sin paréntesis.

**4. ¿Dapper genera migraciones?**
No. Se combina con DbUp, FluentMigrator, scripts versionados o las migraciones de EF Core.

-----

## Práctica

**Ejercicio 1.** Escribe un método que busque tareas cuyo título contenga un texto, de forma segura y sin distinguir mayúsculas.

<details>
<summary>Solución</summary>

```csharp
public static Task<IEnumerable<Tarea>> BuscarAsync(DbConnection conexion, string texto, CancellationToken ct) =>
    conexion.QueryAsync<Tarea>(new CommandDefinition(
        """
        SELECT id, titulo, descripcion, completada, categoria_id
        FROM tareas
        WHERE lower(titulo) LIKE '%' || lower(@Texto) || '%'
        ORDER BY id
        """,
        new { Texto = texto },
        cancellationToken: ct));
```

El texto del usuario viaja como parámetro; los `%` los arma el propio SQL con `||` (concatenación de SQLite, `+` en SQL Server). Si el usuario escribe `%` o `_`, actúan como comodines: para tratarlos como literales hay que escaparlos y usar `ESCAPE`.

</details>

**Ejercicio 2.** Este código devuelve tareas con `CategoriaId = 0` aunque en la base todas tienen categoría. ¿Por qué?

```csharp
var tareas = await conexion.QueryAsync<Tarea>("SELECT id, titulo, categoria_id FROM tareas");
```

<details>
<summary>Solución</summary>

La columna se llama `categoria_id` y la propiedad `CategoriaId`. Sin `MatchNamesWithUnderscores = true`, Dapper no las relaciona, ignora la columna y deja la propiedad en `0`, sin error. Arreglos: `SELECT categoria_id AS CategoriaId`, o activar `DefaultTypeMap.MatchNamesWithUnderscores = true` al iniciar la aplicación.

</details>

-----

## Siguiente lección

[Dapper avanzado y convivencia con EF Core](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md)
