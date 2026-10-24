# Índices y planes de ejecución

## En una frase

Un índice es una estructura ordenada que permite a la base encontrar filas sin recorrer toda la tabla; el **plan de ejecución** muestra si una consulta lo usa o no, y la diferencia entre "busca" y "recorre" es la diferencia entre milisegundos y segundos cuando la tabla crece.

-----

## Antes de empezar

Conviene que ya sepas:

* `HasIndex` y el modelado con Fluent API: [Modelado con convenciones, atributos y Fluent API](../01-entity-framework-core/03-Modelado%20con%20convenciones%20atributos%20y%20Fluent%20API.md).
* Ver el SQL que genera EF Core (`ToQueryString`): [Migraciones y consultas](../01-entity-framework-core/04-Migraciones%20y%20consultas.md).
* ADO.NET para ejecutar SQL propio: [ADO.NET a fondo](../02-ado-net-y-dapper/01-ADO%20NET%20a%20fondo.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Índice:** estructura ordenada (normalmente un árbol B) con los valores de una o más columnas y la ubicación de cada fila.
* **Plan de ejecución:** la estrategia que elige el motor para resolver una consulta: qué índices usa, en qué orden une tablas, si ordena en memoria.
* **Scan frente a seek / search:** recorrer toda la tabla (o todo un índice) frente a ir directo a las filas que cumplen la condición.
* **Índice compuesto:** índice sobre varias columnas, ordenado primero por la primera, después por la segunda, etc.
* **Índice que cubre (*covering index*):** índice que contiene todas las columnas que la consulta necesita, así que no hace falta leer la tabla.
* **Sargable:** condición que el motor puede resolver con un índice (*Search ARGument ABLE*).
* **Selectividad:** proporción de filas que una condición deja pasar; cuanto menor, más útil es el índice.

-----

## El problema

La tabla `Tareas` tiene 3 millones de filas. El dashboard tarda 8 segundos en cargar. Sus consultas:

```csharp
// 1. Búsqueda por título, sin distinguir mayúsculas
db.Tareas.Where(t => t.Titulo.ToLower() == texto.ToLower())

// 2. Pendientes de una categoría
db.Tareas.Where(t => t.CategoriaId == categoriaId && !t.Completada)

// 3. Las más prioritarias
db.Tareas.OrderByDescending(t => t.Prioridad).Take(20)
```

Alguien agrega un índice por cada columna "para que vaya rápido": `Titulo`, `CategoriaId`, `Completada`, `Prioridad`. El dashboard sigue lento, y ahora las inserciones también:

1. La búsqueda por título no usa el índice: `lower(Titulo)` **transforma** la columna, y el índice está ordenado por `Titulo`, no por `lower(Titulo)`.
2. Para las pendientes de una categoría, el motor usa el índice de `CategoriaId` y después revisa una por una las 200 000 tareas de esa categoría para filtrar `Completada`. El índice sobre `Completada` sola es casi inútil: con dos valores posibles, no es selectivo.
3. Cada `INSERT` ahora actualiza cinco árboles (la tabla y cuatro índices).

Nadie miró un **plan de ejecución**: se agregaron índices a ciegas.

Las consecuencias:

* consultas lentas que no mejoran aunque "haya índices";
* escrituras más lentas y más espacio en disco por índices que nadie usa;
* tiempo perdido optimizando sin medir.

-----

## Cómo funciona

### 1. Qué es un índice

```text
Tabla Tareas (orden de inserción)        Índice IX_Tareas_Titulo (ordenado por Titulo)
┌────┬───────────────┬──────┐            ┌───────────────┬──────┐
│ Id │ Titulo        │ ...  │            │ Titulo        │ Fila │
├────┼───────────────┼──────┤            ├───────────────┼──────┤
│ 1  │ Pagar luz     │      │            │ Barrer        │ 3    │
│ 2  │ Revisar PR    │      │◄───────────│ Pagar luz     │ 1    │   buscar "Pagar luz":
│ 3  │ Barrer        │      │            │ Revisar PR    │ 2    │   O(log n) en el árbol
└────┴───────────────┴──────┘            └───────────────┴──────┘   en lugar de O(n) en la tabla
```

En una tabla de 3 millones de filas, un árbol B encuentra un valor en unas pocas lecturas de páginas, en lugar de leer las 3 millones de filas.

### 2. Leer un plan de ejecución

| Motor | Cómo verlo | "Busca" | "Recorre" |
| --- | --- | --- | --- |
| SQL Server | Plan de ejecución en SSMS / `SET STATISTICS IO ON` | `Index Seek` | `Table Scan`, `Clustered Index Scan` |
| PostgreSQL | `EXPLAIN ANALYZE SELECT ...` | `Index Scan`, `Index Only Scan`, `Bitmap Index Scan` | `Seq Scan` |
| SQLite | `EXPLAIN QUERY PLAN SELECT ...` | `SEARCH ... USING INDEX` | `SCAN` |

`EXPLAIN ANALYZE` (PostgreSQL) ejecuta la consulta y muestra tiempos y filas reales de cada paso; compara las filas **estimadas** con las **reales**: una diferencia grande indica estadísticas desactualizadas.

### 3. Sargable: no transformes la columna

| ❌ No sargable | ✔ Sargable |
| --- | --- |
| `WHERE lower(Titulo) = 'pagar luz'` | `WHERE Titulo = 'Pagar luz'` con una intercalación (*collation*) que no distinga mayúsculas |
| `WHERE YEAR(Fecha) = 2026` | `WHERE Fecha >= '2026-01-01' AND Fecha < '2027-01-01'` |
| `WHERE Titulo LIKE '%luz'` | `WHERE Titulo LIKE 'Pagar%'` (prefijo; en SQL Server o PostgreSQL con el operador adecuado) |
| `WHERE Total * 1.21 > 1000` | `WHERE Total > 1000 / 1.21` |

Regla: la columna sola a un lado del operador. Si necesitas buscar por una expresión, muchos motores permiten **indexar la expresión** (columna calculada en SQL Server, índice de expresión en PostgreSQL y SQLite).

### 4. Índices compuestos: el orden importa

Un índice sobre `(CategoriaId, Completada)` está ordenado primero por categoría y, dentro de cada categoría, por estado:

| Condición | ¿Usa el índice? | Por qué |
| --- | --- | --- |
| `CategoriaId = 3` | ✔ | Es el prefijo del índice |
| `CategoriaId = 3 AND Completada = 0` | ✔ (mejor) | Usa las dos columnas |
| `Completada = 0` | ❌ (en general) | Sin la primera columna, los valores de la segunda están dispersos |
| `CategoriaId = 3 ORDER BY Completada` | ✔ sin ordenar en memoria | Dentro de la categoría ya están ordenados |

Orden recomendado: primero las columnas de **igualdad**, después las de **rango** u **ordenamiento**.

### 5. Índices que cubren la consulta

Si el índice contiene todas las columnas que la consulta necesita, el motor no lee la tabla:

```csharp
// SQL Server y PostgreSQL: columnas incluidas (no ordenan, solo acompañan)
modelBuilder.Entity<Tarea>()
    .HasIndex(t => new { t.CategoriaId, t.Completada })
    .IncludeProperties(t => t.Titulo);
```

`SELECT Titulo FROM Tareas WHERE CategoriaId = 3 AND Completada = 0` se resuelve leyendo solo el índice (*Index Only Scan* en PostgreSQL, sin *Key Lookup* en SQL Server). Es otra razón para **proyectar** solo las columnas necesarias: un `SELECT *` nunca está cubierto.

### 6. Otros tipos útiles

```csharp
builder.HasIndex(t => t.Email).IsUnique();                                // unicidad + búsqueda
builder.HasIndex(t => t.CategoriaId).HasFilter("[Completada] = 0");       // filtrado (SQL Server; en PostgreSQL, "\"Completada\" = false")
```

* **Único:** garantiza la regla y acelera la búsqueda.
* **Filtrado / parcial:** indexa solo un subconjunto (las pendientes); más chico y más rápido si las consultas siempre filtran por esa condición.
* **Claves foráneas:** EF Core crea un índice por cada FK por convención. Sin él, cada `JOIN` y cada borrado en cascada recorrería la tabla hija.

### 7. Cada índice cuesta

| Gana | Paga |
| --- | --- |
| Lecturas que filtran u ordenan por esas columnas | Cada `INSERT`, `UPDATE` (de esas columnas) y `DELETE` actualiza el índice |
| | Espacio en disco y en memoria (caché de páginas) |
| | Más opciones para el optimizador, que a veces elige mal |

Crea índices para las consultas **reales** (las que aparecen en los logs o en el *query store*), revisa el plan antes y después, y quita los que nadie usa (SQL Server: `sys.dm_db_index_usage_stats`; PostgreSQL: `pg_stat_user_indexes`).

-----

## Ejemplo completo

Consola que crea 1000 tareas con EF Core y pide a SQLite el plan de varias consultas. `Indices.csproj` con `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0. `Program.cs`:

```csharp
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

var opciones = new DbContextOptionsBuilder<TareasContext>().UseSqlite("Data Source=indices.db").Options;

await using (var db = new TareasContext(opciones))
{
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();
    db.Tareas.AddRange(Enumerable.Range(1, 1000).Select(i => new Tarea
    {
        Titulo = $"Tarea {i}",
        CategoriaId = i % 10,
        Completada = i % 3 == 0,
        Prioridad = i % 5
    }));
    await db.SaveChangesAsync();
}

await using var conexion = new SqliteConnection("Data Source=indices.db");
await conexion.OpenAsync();

(string Caso, string Sql)[] consultas =
[
    ("Igualdad sobre columna indexada", "SELECT Id, Titulo, Completada FROM Tareas WHERE Titulo = 'Tarea 500'"),
    ("Función sobre la columna", "SELECT Id, Titulo, Completada FROM Tareas WHERE lower(Titulo) = 'tarea 500'"),
    ("LIKE con comodín al inicio", "SELECT Id, Titulo, Completada FROM Tareas WHERE Titulo LIKE '%500'"),
    ("Prefijo del índice compuesto", "SELECT Id, Titulo, Completada FROM Tareas WHERE CategoriaId = 3"),
    ("Índice compuesto completo", "SELECT Id, Titulo, Completada FROM Tareas WHERE CategoriaId = 3 AND Completada = 0"),
    ("Solo la segunda columna", "SELECT Id, Titulo, Completada FROM Tareas WHERE Completada = 0"),
    ("Índice que cubre la consulta", "SELECT COUNT(*) FROM Tareas WHERE CategoriaId = 3"),
    ("ORDER BY sin índice", "SELECT Id, Titulo FROM Tareas ORDER BY Prioridad")
];

foreach (var (caso, sql) in consultas)
{
    Console.WriteLine($"{caso,-32} {await PlanAsync(sql)}");
}

async Task<string> PlanAsync(string sql)
{
    await using var cmd = conexion.CreateCommand();
    cmd.CommandText = "EXPLAIN QUERY PLAN " + sql;
    await using var reader = await cmd.ExecuteReaderAsync();

    var pasos = new List<string>();
    while (await reader.ReadAsync())
    {
        pasos.Add(reader.GetString(3));   // columna "detail"
    }
    return string.Join(" | ", pasos);
}

public class Tarea
{
    public int Id { get; set; }
    public required string Titulo { get; set; }
    public int CategoriaId { get; set; }
    public bool Completada { get; set; }
    public int Prioridad { get; set; }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Tarea> Tareas => Set<Tarea>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Tarea>(tarea =>
        {
            tarea.HasIndex(t => t.Titulo);
            tarea.HasIndex(t => new { t.CategoriaId, t.Completada });
        });
    }
}
```

Salida (con la versión de SQLite que incluye `Microsoft.Data.Sqlite` 10; versiones de SQLite anteriores a la 3.36 escriben `SCAN TABLE` en lugar de `SCAN`):

```text
Igualdad sobre columna indexada  SEARCH Tareas USING INDEX IX_Tareas_Titulo (Titulo=?)
Función sobre la columna         SCAN Tareas
LIKE con comodín al inicio       SCAN Tareas
Prefijo del índice compuesto     SEARCH Tareas USING INDEX IX_Tareas_CategoriaId_Completada (CategoriaId=?)
Índice compuesto completo        SEARCH Tareas USING INDEX IX_Tareas_CategoriaId_Completada (CategoriaId=? AND Completada=?)
Solo la segunda columna          SCAN Tareas
Índice que cubre la consulta     SEARCH Tareas USING COVERING INDEX IX_Tareas_CategoriaId_Completada (CategoriaId=?)
ORDER BY sin índice              SCAN Tareas | USE TEMP B-TREE FOR ORDER BY
```

Qué observar:

* La misma columna `Titulo` con índice da `SEARCH` en la igualdad y `SCAN` cuando se le aplica `lower()` o un `LIKE '%...'`: el índice existe, pero la consulta no puede usarlo.
* El índice compuesto sirve para su prefijo (`CategoriaId`) y para las dos columnas, pero no para la segunda sola.
* `COUNT(*)` por categoría usa un **COVERING INDEX**: todo lo que necesita está en el índice y no lee la tabla.
* `ORDER BY Prioridad` sin índice obliga a construir un árbol temporal para ordenar (`USE TEMP B-TREE`): con millones de filas, ese ordenamiento es costoso.
* `HasIndex` generó los nombres `IX_Tareas_Titulo` e `IX_Tareas_CategoriaId_Completada` por convención.
* Con 1000 filas todo es instantáneo; el plan, en cambio, es el mismo que con 3 millones. Por eso se mira el plan, no solo el tiempo en desarrollo.

-----

## Errores comunes

**1. Funciones sobre columnas indexadas.**
Qué pasa: `SCAN` completo aunque haya índice (`ToLower()`, `YEAR()`, `Trim()` en el `Where`).
Por qué: el índice está ordenado por la columna, no por la expresión.
Arreglo: comparar la columna directa (y resolver mayúsculas con la *collation*), rangos en lugar de funciones de fecha, o indexar la expresión.

**2. Un índice por columna "por las dudas".**
Qué pasa: escrituras más lentas y espacio desperdiciado, sin mejorar las consultas que combinan columnas.
Por qué: los índices de una columna no reemplazan a un compuesto bien ordenado.
Arreglo: índices diseñados para las consultas reales, revisando el plan.

**3. Índice compuesto con el orden equivocado.**
Qué pasa: `(Completada, CategoriaId)` para consultas que filtran solo por `CategoriaId`: el índice no se usa.
Por qué: solo el prefijo del índice está ordenado de forma utilizable.
Arreglo: primero las columnas de igualdad que siempre aparecen, después rangos y ordenamientos.

**4. Índices sobre columnas poco selectivas.**
Qué pasa: un índice sobre `Completada` (dos valores) que el optimizador ignora o que empeora la consulta.
Por qué: si la condición deja pasar media tabla, recorrerla es más barato que saltar del índice a la tabla.
Arreglo: combinar con columnas selectivas, o un índice filtrado si siempre se consulta el valor minoritario.

**5. Optimizar mirando solo el tiempo en desarrollo.**
Qué pasa: "en mi máquina tarda 2 ms" y en producción tarda 4 segundos.
Por qué: con pocas filas cualquier plan es rápido.
Arreglo: mirar el plan (y probar con volúmenes realistas).

**6. `SELECT *` que impide cubrir la consulta.**
Qué pasa: búsquedas por índice seguidas de miles de lecturas a la tabla (*Key Lookup*).
Por qué: el índice no tiene todas las columnas pedidas.
Arreglo: proyectar solo lo necesario y, si hace falta, `IncludeProperties`.

-----

## Según la versión de .NET

* **EF Core 1.0:** `HasIndex`, índices únicos e índices automáticos sobre claves foráneas.
* **EF Core 5:** atributo `[Index]`, índices filtrados (`HasFilter`) e `IncludeProperties` (SQL Server y Npgsql).
* **EF Core 7:** orden descendente por columna en índices compuestos (`IsDescending`).
* **SQLite 3.36 (2021):** el plan pasa a mostrar `SCAN tabla` en lugar de `SCAN TABLE tabla`.

-----

## Cuándo sí y cuándo no

**Crea un índice cuando:** una consulta frecuente filtra, une u ordena por esas columnas y el plan muestra un recorrido completo sobre una tabla grande.

**Usa un índice compuesto cuando:** las consultas combinan siempre las mismas columnas; ordénalo por igualdad primero.

**No crees un índice cuando:** la tabla es chica, la columna es poco selectiva, la tabla recibe muchísimas escrituras y pocas lecturas por esa columna, o nadie consulta así (verifícalo en las estadísticas de uso).

-----

## Resumen en 5 líneas

1. Un índice permite buscar en O(log n) en lugar de recorrer la tabla.
2. El plan de ejecución dice si la consulta busca (`SEARCH`/`Seek`) o recorre (`SCAN`).
3. Condiciones sargables: la columna sola, sin funciones ni comodines al inicio.
4. Índices compuestos: el prefijo manda; igualdad primero, rango y orden después.
5. Cada índice acelera lecturas y encarece escrituras: créalos para consultas reales, midiendo.

-----

## Para profundizar

<details>
<summary>Índice agrupado (clustered) en SQL Server</summary>

En SQL Server, la tabla misma suele estar ordenada por un índice agrupado (por defecto, la clave primaria). Los índices no agrupados apuntan a la clave del agrupado. Por eso un `Guid` aleatorio como clave agrupada fragmenta la tabla (cada inserción cae en cualquier página), mientras que un `int IDENTITY` o un Guid secuencial inserta siempre al final. En PostgreSQL las tablas no están agrupadas por defecto (son *heaps*), y el problema se ve menos.

</details>

<details>
<summary>Estadísticas y parámetros</summary>

El optimizador elige el plan con **estadísticas** sobre la distribución de los datos. Si están desactualizadas, puede elegir un recorrido cuando convenía un índice. PostgreSQL las actualiza con `ANALYZE` (y el *autovacuum*), SQL Server con `UPDATE STATISTICS`. En SQL Server existe además el *parameter sniffing*: el plan se compila con el primer valor de un parámetro y se reutiliza para otros que quizás necesitaban otro plan.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un índice es como el índice de un libro: permite encontrar filas sin recorrer toda la tabla. Se crea con `HasIndex` en EF Core. Para saber si una consulta lo usa, miro el plan de ejecución: "seek" o "search" significa que lo usa, "scan" que recorre todo.

### Respuesta ampliada (semi-senior)

Diseño índices a partir de las consultas reales y verifico el plan antes y después (`EXPLAIN ANALYZE`, plan de SSMS). Cuido que las condiciones sean sargables (sin funciones sobre la columna ni comodines al inicio), ordeno los compuestos con las columnas de igualdad primero y uso columnas incluidas para cubrir consultas frecuentes, lo que también exige proyectar solo lo necesario. Considero índices filtrados para subconjuntos consultados siempre igual y evito columnas poco selectivas. Sé que cada índice encarece las escrituras, así que reviso las estadísticas de uso y quito los que sobran.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `WHERE lower(Email) = ...` no usa el índice?**
Porque el índice está ordenado por `Email`, no por `lower(Email)`.

**2. ¿Un índice sobre (A, B) sirve para filtrar por B?**
En general no: solo el prefijo (A, o A y B) está ordenado de forma utilizable.

**3. ¿Qué es un índice que cubre la consulta?**
Uno que contiene todas las columnas necesarias, así que no hace falta leer la tabla.

**4. ¿Por qué no indexar todas las columnas?**
Porque cada índice encarece las escrituras y ocupa espacio, y muchos no se usarían.

-----

## Práctica

**Ejercicio 1.** Reescribe estas consultas para que sean sargables, suponiendo índices sobre `Fecha` y sobre `Email`.

```csharp
db.Pedidos.Where(p => p.Fecha.Year == 2026 && p.Fecha.Month == 3)
db.Clientes.Where(c => c.Email.Trim().ToLower() == email.Trim().ToLower())
```

<details>
<summary>Solución</summary>

```csharp
var desde = new DateTime(2026, 3, 1);
var hasta = desde.AddMonths(1);
db.Pedidos.Where(p => p.Fecha >= desde && p.Fecha < hasta)

string normalizado = email.Trim().ToLowerInvariant();
db.Clientes.Where(c => c.Email == normalizado)
```

La fecha pasa a un rango sobre la columna sola. Para el email, la normalización se hace **al guardar** (el email se almacena ya recortado y en minúsculas) y al buscar se normaliza solo el valor del parámetro. Alternativa: una intercalación que no distinga mayúsculas en la columna.

</details>

**Ejercicio 2.** El listado más usado es "pendientes de una categoría, ordenadas por prioridad descendente, mostrando título y prioridad". Diseña el índice en EF Core para SQL Server.

<details>
<summary>Solución</summary>

```csharp
modelBuilder.Entity<Tarea>()
    .HasIndex(t => new { t.CategoriaId, t.Completada, t.Prioridad })
    .IsDescending(false, false, true)
    .IncludeProperties(t => t.Titulo);
```

* `CategoriaId` y `Completada` (igualdad) primero; `Prioridad` (orden) después.
* `IsDescending` hace que el índice ya esté en el orden del `ORDER BY`, sin ordenar en memoria.
* `Titulo` incluida: la consulta `SELECT Titulo, Prioridad ... WHERE CategoriaId = @c AND Completada = 0 ORDER BY Prioridad DESC` queda cubierta.

Si casi siempre se consultan solo las pendientes, un índice filtrado (`HasFilter("[Completada] = 0")`) sobre `(CategoriaId, Prioridad)` es más chico todavía.

</details>

-----

## Siguiente lección

[Detectar consultas lentas en producción](03-Detectar%20consultas%20lentas%20en%20produccion.md)
