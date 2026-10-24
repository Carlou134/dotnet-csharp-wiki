# Paginación y consultas eficientes

## En una frase

Paginar con `Skip`/`Take` (*offset*) es simple pero se vuelve lento en páginas profundas y repite u omite filas cuando los datos cambian entre páginas; la paginación por **cursor** (*keyset*) pide "los siguientes después de este" usando un índice y no tiene esos problemas, y junto con proyecciones, consultas compiladas y *pooling* forma el kit básico para que el acceso a datos aguante carga.

-----

## Antes de empezar

Conviene que ya sepas:

* Proyecciones y `AsNoTracking`: [Migraciones y consultas](../01-entity-framework-core/04-Migraciones%20y%20consultas.md).
* Índices compuestos y planes: [Índices y planes de ejecución](02-Indices%20y%20planes%20de%20ejecucion.md).
* Contar consultas con interceptores: [Detectar consultas lentas en producción](03-Detectar%20consultas%20lentas%20en%20produccion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Paginación por offset:** pedir una página saltando N filas (`Skip(N).Take(tamaño)`, `OFFSET`/`LIMIT`).
* **Paginación por cursor (*keyset*):** pedir las filas que siguen a la última vista, filtrando por su clave de ordenamiento (`WHERE Id < @ultimo`).
* **Cursor:** valor (normalmente opaco para el cliente) que identifica dónde terminó la página anterior.
* **Orden total:** ordenamiento sin empates, logrado agregando una columna única (como `Id`) al final.
* **Consulta compilada:** consulta LINQ traducida una vez y reutilizada (`EF.CompileAsyncQuery`), sin volver a procesar el árbol de expresión.
* **Pooling de contextos:** reutilizar instancias de `DbContext` entre peticiones (`AddDbContextPool`).

-----

## El problema

Un feed de tareas, de la más nueva a la más vieja, con 20 por página:

```csharp
app.MapGet("/tareas", async (int pagina, AppDbContext db) =>
{
    var items = await db.Tareas.OrderByDescending(t => t.Id)
        .Skip((pagina - 1) * 20).Take(20).ToListAsync();
    var total = await db.Tareas.CountAsync();
    return new { items, total };
});
```

Con 5 millones de filas:

1. **Las páginas profundas son lentas.** `GET /tareas?pagina=200000` hace que la base genere y descarte 3 999 980 filas para devolver 20. La página 1 tarda 5 ms; la 200 000, varios segundos.
2. **El `COUNT(*)` de toda la tabla en cada petición** cuesta más que la propia página, para mostrar un número que nadie necesita exacto.
3. **Filas repetidas y perdidas.** Un usuario está en la página 1. Mientras la lee, se crean 3 tareas nuevas. Pide la página 2: las 3 últimas de la página 1 aparecen **otra vez** arriba, porque todo se corrió 3 lugares. Si en cambio se borran filas, otras quedan **sin mostrarse nunca**.
4. **Orden inestable.** Otra pantalla ordena por `FechaCreacion`, que tiene empates (muchas tareas en el mismo segundo). Sin desempate, la base puede devolver las empatadas en cualquier orden en cada consulta: la misma fila aparece en dos páginas y otra en ninguna.

Las consecuencias:

* tiempos que crecen con el número de página;
* consultas de conteo caras en cada petición;
* listados que repiten u omiten datos, sobre todo en *scroll* infinito.

-----

## Cómo funciona

### 1. Offset frente a cursor

```text
OFFSET (Skip 40, Take 20)                     CURSOR (después del Id 941)
┌─────────────────────────────┐               ┌─────────────────────────────┐
│ fila 1 ... fila 40  descartadas│              │ índice: salta directo al 940  │
│ fila 41 ... fila 60 devueltas  │              │ fila 940 ... fila 921 devueltas│
└─────────────────────────────┘               └─────────────────────────────┘
costo ∝ offset + tamaño                        costo ∝ tamaño (con índice)
```

| | Offset (`Skip`/`Take`) | Cursor (*keyset*) |
| --- | --- | --- |
| Costo en página profunda | Crece con el número de página | Constante con un índice adecuado |
| Datos que cambian entre páginas | Repite u omite filas | Estable: "lo que sigue a X" |
| Saltar a la página 37 | ✔ | ❌ (solo siguiente/anterior) |
| Implementación | Trivial | Requiere orden total y un cursor |
| Ideal para | Tablas chicas, paneles con números de página | Feeds, *scroll* infinito, APIs públicas, exportaciones |

### 2. Cursor con una sola columna

```csharp
var pagina = await db.Tareas
    .Where(t => ultimoId == null || t.Id < ultimoId)   // "después" del último visto
    .OrderByDescending(t => t.Id)
    .Take(tamanio)
    .Select(t => new TareaDto(t.Id, t.Titulo))
    .ToListAsync(ct);
```

El `WHERE Id < @ultimo ORDER BY Id DESC` se resuelve con el índice de la clave primaria: la base salta directo al punto y lee `tamanio` filas.

### 3. Cursor con varias columnas: orden total

Para ordenar por fecha (que tiene empates), se agrega el `Id` como desempate y el filtro compara la tupla:

```csharp
// ORDER BY FechaCreacion DESC, Id DESC
// "después de (fecha, id)" = fecha anterior, o la misma fecha con id menor
.Where(t => t.FechaCreacion < fecha || (t.FechaCreacion == fecha && t.Id < id))
.OrderByDescending(t => t.FechaCreacion).ThenByDescending(t => t.Id)
```

Necesita un índice compuesto `(FechaCreacion DESC, Id DESC)` para ser eficiente. Sin el desempate, las filas con la misma fecha pueden repetirse o perderse en el corte entre páginas.

### 4. El cursor en la API

```json
{
  "items": [ { "id": 940, "titulo": "..." }, ... ],
  "siguiente": "eyJmIjoiMjAyNi0wMy0xNVQxMjowMDowMCIsImkiOjkyMX0"
}
```

* El cursor es **opaco**: un Base64 de `{fecha, id}` del último elemento. El cliente lo devuelve tal cual (`GET /tareas?despues=eyJm...`) y no depende de su formato, así que puedes cambiarlo sin romper clientes.
* Para saber si hay más sin un `COUNT`, pide `tamanio + 1` filas: si vuelven todas, hay otra página (y descartas la extra).
* Si el total es necesario, que sea aproximado o se calcule aparte (y quizás se cachee): rara vez hace falta exacto en cada página.

### 5. Consultas compiladas

```csharp
private static readonly Func<AppDbContext, int, int, IAsyncEnumerable<TareaDto>> SiguientesDespuesDe =
    EF.CompileAsyncQuery((AppDbContext db, int ultimoId, int tamanio) =>
        db.Tareas.Where(t => t.Id < ultimoId)
                 .OrderByDescending(t => t.Id)
                 .Take(tamanio)
                 .Select(t => new TareaDto(t.Id, t.Titulo)));

await foreach (var tarea in SiguientesDespuesDe(db, 941, 20)) { ... }
```

EF Core ya guarda en caché la traducción de cada consulta, pero igual tiene que recorrer el árbol de expresión en cada ejecución para buscarla. Una consulta compilada se salta ese paso. La ganancia es de microsegundos por ejecución: importa en rutas que se ejecutan miles de veces por segundo, no en un listado normal. Mide antes de compilarlo todo.

### 6. El resto del kit

| Técnica | Qué evita | Lección |
| --- | --- | --- |
| Proyección (`Select` a DTO) | Columnas de más y tracking | [Migraciones y consultas](../01-entity-framework-core/04-Migraciones%20y%20consultas.md) |
| `AsNoTracking` | Costo del change tracker en lecturas | Ídem |
| `AsSplitQuery` | Explosión cartesiana con varias colecciones | [Relaciones](../01-entity-framework-core/05-Relaciones%20y%20carga%20de%20datos.md) |
| `ExecuteUpdate` / `ExecuteDelete` | Cargar entidades para cambios masivos | [CRUD en APIs](../01-entity-framework-core/06-CRUD%20en%20APIs%20con%20EF%20Core.md) |
| Un solo `SaveChanges` con muchos cambios | Una transacción e ida por cambio (EF Core agrupa en lotes) | Ídem |
| `AddDbContextPool` | Crear y configurar un contexto nuevo por petición | [DbContext](../01-entity-framework-core/02-DbContext%20entidades%20y%20configuracion.md) |
| `AsAsyncEnumerable` / streaming | Cargar millones de filas en memoria para exportar | — |

-----

## Ejemplo completo

Consola que compara offset y cursor cuando se inserta una fila entre una página y la siguiente. `Paginacion.csproj` con `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0. `Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

var opciones = new DbContextOptionsBuilder<TareasContext>().UseSqlite("Data Source=paginacion.db").Options;
const int Tamanio = 3;

// 1. Offset: la inserción corre todo un lugar
await Reiniciar();
await using (var db = new TareasContext(opciones))
{
    Console.WriteLine($"Offset  página 1: {Ids(await PaginaOffset(db, numero: 1))}");
    await InsertarTarea(db);
    Console.WriteLine($"Offset  página 2: {Ids(await PaginaOffset(db, numero: 2))}   <- la 8 se repite");
}

// 2. Cursor: "lo que sigue a la última vista" no cambia
await Reiniciar();
await using (var db = new TareasContext(opciones))
{
    var (pagina1, _) = await PaginaCursor(db, despuesDe: null);
    Console.WriteLine($"Cursor  página 1: {Ids(pagina1)}");
    await InsertarTarea(db);
    var (pagina2, _) = await PaginaCursor(db, despuesDe: pagina1[^1].Id);
    Console.WriteLine($"Cursor  página 2: {Ids(pagina2)}");

    var (ultima, hayMas) = await PaginaCursor(db, despuesDe: 2);
    Console.WriteLine($"Cursor  después de 2: {Ids(ultima)} (hay más: {hayMas})");

    // 3. La misma consulta, compilada
    var compilada = new List<TareaFila>();
    await foreach (var tarea in Consultas.SiguientesDespuesDe(db, 5, Tamanio))
    {
        compilada.Add(tarea);
    }
    Console.WriteLine($"Compilada después de 5: {Ids(compilada)}");
}

static Task<List<TareaFila>> PaginaOffset(TareasContext db, int numero) =>
    db.Tareas.AsNoTracking()
        .OrderByDescending(t => t.Id)
        .Skip((numero - 1) * Tamanio)
        .Take(Tamanio)
        .Select(t => new TareaFila(t.Id, t.Titulo))
        .ToListAsync();

static async Task<(List<TareaFila> Items, bool HayMas)> PaginaCursor(TareasContext db, int? despuesDe)
{
    List<TareaFila> filas = await db.Tareas.AsNoTracking()
        .Where(t => despuesDe == null || t.Id < despuesDe)
        .OrderByDescending(t => t.Id)
        .Take(Tamanio + 1)                       // una de más para saber si hay otra página
        .Select(t => new TareaFila(t.Id, t.Titulo))
        .ToListAsync();

    bool hayMas = filas.Count > Tamanio;
    return (filas.Take(Tamanio).ToList(), hayMas);
}

static async Task InsertarTarea(TareasContext db)
{
    db.Tareas.Add(new Tarea { Titulo = "Tarea nueva" });
    await db.SaveChangesAsync();
    Console.WriteLine("        (se inserta una tarea nueva)");
}

static string Ids(IEnumerable<TareaFila> filas) => string.Join(", ", filas.Select(f => f.Id));

async Task Reiniciar()
{
    await using var db = new TareasContext(opciones);
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();
    db.Tareas.AddRange(Enumerable.Range(1, 10).Select(i => new Tarea { Titulo = $"Tarea {i}" }));
    await db.SaveChangesAsync();
}

public static class Consultas
{
    public static readonly Func<TareasContext, int, int, IAsyncEnumerable<TareaFila>> SiguientesDespuesDe =
        EF.CompileAsyncQuery((TareasContext db, int ultimoId, int tamanio) =>
            db.Tareas.Where(t => t.Id < ultimoId)
                     .OrderByDescending(t => t.Id)
                     .Take(tamanio)
                     .Select(t => new TareaFila(t.Id, t.Titulo)));
}

public sealed record TareaFila(int Id, string Titulo);

public class Tarea
{
    public int Id { get; set; }
    public required string Titulo { get; set; }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Tarea> Tareas => Set<Tarea>();
}
```

Salida:

```text
Offset  página 1: 10, 9, 8
        (se inserta una tarea nueva)
Offset  página 2: 8, 7, 6   <- la 8 se repite
Cursor  página 1: 10, 9, 8
        (se inserta una tarea nueva)
Cursor  página 2: 7, 6, 5
Cursor  después de 2: 1 (hay más: False)
Compilada después de 5: 4, 3, 2
```

Qué observar:

* Con offset, la tarea nueva (id 11) empujó todo un lugar: la página 2 empezó con la 8, que ya se había mostrado. En un *scroll* infinito, el usuario ve un elemento duplicado.
* Con cursor, la página 2 es "los siguientes después del 8": la inserción no la afecta.
* Pedir `Tamanio + 1` filas permite responder "hay más" sin un `COUNT`: después del 2 solo queda la 1.
* La consulta compilada devuelve lo mismo que la normal; cambia el costo de preparar la consulta, no el SQL.
* `TareaFila` es un record posicional: EF Core lo construye con su constructor en la proyección (a diferencia de Dapper con SQLite, aquí los tipos los convierte EF Core).

-----

## Errores comunes

**1. Paginar sin `OrderBy`.**
Qué pasa: páginas con filas repetidas o faltantes y la advertencia de EF Core sobre `Skip`/`Take` sin orden.
Por qué: SQL no garantiza ningún orden sin `ORDER BY`.
Arreglo: ordenar siempre, y por una clave única al final.

**2. Ordenar por una columna con empates sin desempate.**
Qué pasa: la misma fila en dos páginas y otra en ninguna.
Por qué: el orden entre empatados no está definido y puede cambiar entre consultas.
Arreglo: `ThenBy(t => t.Id)` (o la clave única que corresponda).

**3. Offset en páginas profundas o feeds.**
Qué pasa: tiempos que crecen con el número de página; duplicados al insertar.
Por qué: la base recorre y descarta todas las filas anteriores.
Arreglo: paginación por cursor con un índice que coincida con el orden.

**4. `COUNT(*)` exacto en cada página.**
Qué pasa: el conteo cuesta más que la página.
Por qué: contar millones de filas con filtros es caro.
Arreglo: `Take(tamanio + 1)` para "hay más", totales aproximados o calculados aparte.

**5. Exponer el cursor como datos internos.**
Qué pasa: los clientes construyen cursores a mano y se rompen al cambiar el ordenamiento.
Por qué: el cursor era `?ultimoId=941&ultimaFecha=...`.
Arreglo: un cursor opaco (Base64) que el cliente solo devuelve.

**6. Compilar todas las consultas "por rendimiento".**
Qué pasa: código más difícil de leer sin una mejora medible.
Por qué: la traducción ya está en caché; la ganancia es de microsegundos.
Arreglo: compilar solo rutas calientes medidas.

-----

## Según la versión de .NET

* **EF Core 2.0:** `EF.CompileQuery` / `EF.CompileAsyncQuery` y `AddDbContextPool`.
* **EF Core 3.0:** `AsAsyncEnumerable` para leer resultados en streaming.
* **EF Core 3.x en adelante:** advertencia registrada cuando se usa `Skip`/`Take`/`First` sin `OrderBy` (`RowLimitingOperationWithoutOrderByWarning`).
* **EF Core 6:** *pooling* con fábrica (`AddPooledDbContextFactory`) y mejoras de rendimiento en consultas compiladas.
* **EF Core 9 / 10:** consultas compiladas por adelantado (*precompiled queries*) como funcionalidad experimental para Native AOT.

-----

## Cuándo sí y cuándo no

**Usa offset cuando:** la tabla es chica o los usuarios necesitan saltar a una página numerada (paneles de administración), y las páginas profundas son raras.

**Usa cursor cuando:** es un feed, un *scroll* infinito, una API pública o una exportación por lotes, o la tabla es grande y cambia mientras se pagina.

**Usa consultas compiladas o pooling cuando:** una medición muestra que el costo de preparar consultas o contextos es significativo en una ruta muy frecuente.

-----

## Resumen en 5 líneas

1. Offset (`Skip`/`Take`) se vuelve lento en páginas profundas y repite u omite filas si los datos cambian.
2. Cursor: "las siguientes después de esta", con un índice que coincide con el orden.
3. Orden total siempre: agrega una clave única como desempate.
4. `Take(tamanio + 1)` para saber si hay más sin `COUNT`; el cursor, opaco.
5. Proyecciones, `AsNoTracking`, `ExecuteUpdate`, pooling y consultas compiladas, aplicados donde se midió.

-----

## Para profundizar

<details>
<summary>Página anterior con cursor</summary>

Para ir hacia atrás, se invierte la comparación y el orden (`WHERE Id > @primero ORDER BY Id ASC`), se toman `tamanio` filas y se invierte la lista en memoria antes de devolverla. Por eso muchas APIs con cursor ofrecen solo "siguiente" (y el cliente guarda las páginas que ya vio), o devuelven dos cursores: `siguiente` y `anterior`.

</details>

<details>
<summary>Exportaciones grandes con streaming</summary>

Para exportar 5 millones de filas a CSV, cargar todo con `ToListAsync` agota la memoria. Con `await foreach (var fila in db.Tareas.AsNoTracking().Select(...).AsAsyncEnumerable())` las filas se leen de a una mientras se escriben en el archivo o en la respuesta HTTP. La conexión queda ocupada durante toda la exportación; alternativa: recorrer por cursor en lotes de 10 000, liberando la conexión entre lotes.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Pagino con `OrderBy` + `Skip` + `Take`. Para tablas grandes o feeds prefiero la paginación por cursor: en lugar de saltar filas, pido las que siguen a la última que vi, con `Where(t => t.Id < ultimoId)`.

### Respuesta ampliada (semi-senior)

Uso offset en tablas chicas o paneles con páginas numeradas, y cursor (*keyset*) en feeds, APIs públicas y tablas grandes: el costo no crece con la profundidad y no repite ni omite filas cuando los datos cambian. Garantizo orden total con una clave única como desempate, comparo la tupla en el `Where` y creo el índice compuesto en ese orden. El cursor es opaco y uso `Take(n + 1)` para saber si hay más en lugar de un `COUNT` por página. Para rutas muy calientes, mido y aplico consultas compiladas o `AddDbContextPool`; para exportaciones, streaming con `AsAsyncEnumerable`.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `Skip(100000)` es lento?**
Porque la base tiene que recorrer y descartar esas 100 000 filas antes de devolver la página.

**2. ¿Qué limitación tiene la paginación por cursor?**
No permite saltar directamente a una página numerada.

**3. ¿Cómo sabes si hay otra página sin contar?**
Pidiendo una fila más que el tamaño de página.

**4. ¿Las consultas compiladas cambian el SQL?**
No: evitan el costo de procesar el árbol de expresión en cada ejecución.

-----

## Práctica

**Ejercicio 1.** Implementa un cursor opaco para ordenar por `FechaCreacion DESC, Id DESC`: una función que codifica `(fecha, id)` en Base64 y otra que lo decodifica, y la consulta que usa ambos.

<details>
<summary>Solución</summary>

```csharp
public static class Cursor
{
    public static string Codificar(DateTime fecha, int id) =>
        Convert.ToBase64String(JsonSerializer.SerializeToUtf8Bytes(new CursorDatos(fecha, id)));

    public static CursorDatos? Decodificar(string? cursor)
    {
        if (string.IsNullOrEmpty(cursor)) return null;
        try
        {
            return JsonSerializer.Deserialize<CursorDatos>(Convert.FromBase64String(cursor));
        }
        catch (Exception ex) when (ex is FormatException or JsonException)
        {
            return null;   // cursor inválido: la API debería responder 400
        }
    }
}

public sealed record CursorDatos(DateTime Fecha, int Id);

// Consulta
CursorDatos? despues = Cursor.Decodificar(despuesDe);
var filas = await db.Tareas.AsNoTracking()
    .Where(t => despues == null
                || t.FechaCreacion < despues.Fecha
                || (t.FechaCreacion == despues.Fecha && t.Id < despues.Id))
    .OrderByDescending(t => t.FechaCreacion).ThenByDescending(t => t.Id)
    .Take(tamanio + 1)
    .Select(t => new { t.Id, t.Titulo, t.FechaCreacion })
    .ToListAsync(ct);

string? siguiente = filas.Count > tamanio
    ? Cursor.Codificar(filas[tamanio - 1].FechaCreacion, filas[tamanio - 1].Id)
    : null;
```

El cursor apunta al último elemento **devuelto** (`filas[tamanio - 1]`), no a la fila extra. Necesita el índice `(FechaCreacion DESC, Id DESC)`. En una API pública, conviene además firmar el cursor (HMAC) para que el cliente no pueda fabricarlo.

</details>

**Ejercicio 2.** Un endpoint exporta todas las tareas a CSV con `var todas = await db.Tareas.ToListAsync();` y se queda sin memoria con 5 millones de filas. Reescríbelo con streaming.

<details>
<summary>Solución</summary>

```csharp
app.MapGet("/tareas/exportar", async (AppDbContext db, HttpResponse response, CancellationToken ct) =>
{
    response.ContentType = "text/csv";
    await using var escritor = new StreamWriter(response.Body);
    await escritor.WriteLineAsync("Id,Titulo");

    await foreach (var t in db.Tareas.AsNoTracking()
                       .OrderBy(t => t.Id)
                       .Select(t => new { t.Id, t.Titulo })
                       .AsAsyncEnumerable()
                       .WithCancellation(ct))
    {
        await escritor.WriteLineAsync($"{t.Id},\"{t.Titulo.Replace("\"", "\"\"")}\"");
    }
});
```

Las filas se leen de a una y se escriben directo en la respuesta: la memoria se mantiene constante. La proyección trae solo dos columnas y `AsNoTracking` evita guardar 5 millones de entidades en el change tracker. Las comillas dentro del título se duplican, como exige el formato CSV.

</details>

-----

## Siguiente lección

Terminaste el módulo. Practica con los [Ejercicios del módulo](Ejercicios.md) y vuelve al [índice](README.md).
