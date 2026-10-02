# Filtrar, ordenar y paginar

## En una frase

LINQ tiene operadores para **filtrar** (`Where`, `Distinct`, `OfType`), **ordenar** (`OrderBy`, `ThenBy`), **paginar** (`Skip`, `Take`, `TakeWhile`), **obtener un elemento** (`First`, `Single`, `Last`), **preguntar** (`Any`, `All`, `Contains`) y **proyectar** (`Select`, `SelectMany`), que se combinan en una cadena de lectura clara.

-----

## Antes de empezar

Conviene que ya sepas:

* `Where`, `Select`, ejecución diferida y materialización, de [Introducción a LINQ](01-Introduccion%20a%20LINQ.md).
* Records, de [Structs y records](../05-tipos-avanzados/02-Structs%20y%20records.md), porque los ejemplos los usan como datos.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Predicado:** lambda que devuelve `bool` y decide si un elemento pasa (`x => x.Paginas > 400`).
* **Clave de ordenamiento:** el valor por el que se ordena (`OrderBy(l => l.Titulo)`).
* **Orden estable:** los elementos con la misma clave conservan su orden original.
* **Paginación:** mostrar los resultados por páginas: saltar los anteriores y tomar los de la página actual.
* **Cuantificador:** operador que responde sí o no sobre la secuencia (`Any`, `All`).
* **Aplanar:** convertir una secuencia de secuencias en una sola (`SelectMany`).

-----

## El problema

Tienes un catálogo de libros y una pantalla de búsqueda con filtros, orden y páginas de 10 resultados:

* "Libros de Java con más de 400 páginas, del más nuevo al más viejo, página 3".
* "¿Hay algún libro publicado en 2005?".
* "El primer libro de la categoría Python, si existe".
* "Todas las categorías distintas del catálogo".

Cada una, con bucles, son 10 o 15 líneas con contadores, banderas y listas temporales. Con LINQ, cada una es una cadena de 2 o 3 operadores que se lee como la frase del requisito.

-----

## Cómo funciona

Los ejemplos usan este catálogo:

```csharp
record Libro(string Titulo, int Paginas, DateTime Publicacion, string[] Categorias, string Estado);

var libros = new List<Libro>
{
    new("Clean Code", 464, new(2008, 8, 1), new[] { "Programación", "Java" }, "PUBLISH"),
    new("Java Concurrency", 425, new(2006, 5, 9), new[] { "Java" }, "PUBLISH"),
    new("Python Crash Course", 544, new(2019, 5, 3), new[] { "Python" }, "PUBLISH"),
    new("Effective Java", 412, new(2018, 1, 6), new[] { "Java" }, "PUBLISH"),
    new("Fluent Python", 0, new(2015, 8, 20), new[] { "Python" }, "MEAP"),
    new("Refactoring", 448, new(2018, 11, 19), new[] { "Programación" }, "PUBLISH"),
};
```

### Filtrar

```csharp
var largos = libros.Where(l => l.Paginas > 400);
var deJava = libros.Where(l => l.Categorias.Contains("Java"));
var javaLargos = libros.Where(l => l.Categorias.Contains("Java") && l.Paginas > 420);

var conIndice = libros.Where((l, i) => i % 2 == 0);   // sobrecarga con la posición
```

`Distinct` quita duplicados; `DistinctBy` los quita según una clave:

```csharp
int[] numeros = { 3, 1, 3, 2, 1 };
var unicos = numeros.Distinct();                        // 3, 1, 2

var unoPorAnio = libros.DistinctBy(l => l.Publicacion.Year);   // el primer libro de cada año
```

`OfType<T>` filtra por tipo (útil con colecciones heterogéneas):

```csharp
object[] mezcla = { 1, "dos", 3.0, "cuatro" };
var textos = mezcla.OfType<string>();                   // "dos", "cuatro"
```

### Ordenar

```csharp
var porTitulo = libros.OrderBy(l => l.Titulo);                    // ascendente
var masNuevos = libros.OrderByDescending(l => l.Publicacion);     // descendente

var porCategoriaYPaginas = libros
    .OrderBy(l => l.Categorias[0])           // criterio principal
    .ThenByDescending(l => l.Paginas);       // desempate
```

* Para un segundo criterio se usa `ThenBy`/`ThenByDescending`, **no** otro `OrderBy`: un segundo `OrderBy` reordenaría todo y descartaría el primer criterio.
* `OrderBy` es **estable**: con claves iguales, se conserva el orden original.
* A diferencia de `List.Sort()`, no modifica la colección: devuelve una secuencia nueva.
* Para ordenar valores simples por sí mismos: `numeros.Order()` y `numeros.OrderDescending()` (.NET 7).

### Paginar: `Skip` y `Take`

```csharp
int tamanoPagina = 2;
int pagina = 2;   // la segunda página (empezando en 1)

var paginaActual = libros
    .OrderBy(l => l.Titulo)                  // ¡ordena siempre antes de paginar!
    .Skip((pagina - 1) * tamanoPagina)       // salta las páginas anteriores
    .Take(tamanoPagina);                     // toma las de esta página
```

El **orden** de los operadores importa:

```csharp
libros.Take(4).Skip(2);   // de los 4 primeros, salta 2 → elementos 3 y 4
libros.Skip(2).Take(4);   // salta 2, toma los 4 siguientes → elementos 3, 4, 5 y 6
```

Variantes:

```csharp
var ultimosDos = libros.TakeLast(2);
var sinLosUltimos = libros.SkipLast(2);

int[] lecturas = { 1, 2, 3, 9, 4, 1 };
var hastaPico = lecturas.TakeWhile(n => n < 9);   // 1, 2, 3       (se DETIENE en el primero que no cumple)
var desdePico = lecturas.SkipWhile(n => n < 9);   // 9, 4, 1       (salta mientras se cumple, luego toma todo)
var menoresA5 = lecturas.Where(n => n < 5);       // 1, 2, 3, 4, 1 (revisa TODOS)
```

`TakeWhile` y `Where` se confunden mucho: `Where` revisa todos los elementos; `TakeWhile` corta en el primero que no cumple.

`Chunk` divide en bloques del tamaño indicado (.NET 6):

```csharp
foreach (Libro[] bloque in libros.Chunk(4))
    Console.WriteLine($"Bloque de {bloque.Length}");   // 4, luego 2
```

### Obtener un elemento

```csharp
Libro primero = libros.First();                                        // el primero
Libro primeroPython = libros.First(l => l.Categorias.Contains("Python"));
Libro? primeroRust = libros.FirstOrDefault(l => l.Categorias.Contains("Rust"));   // null si no hay
Libro ultimo = libros.Last();
Libro tercero = libros.ElementAt(2);

Libro unico = libros.Single(l => l.Titulo == "Refactoring");           // exige exactamente uno
```

| Operador | Si no hay ninguno | Si hay más de uno |
| --- | --- | --- |
| `First` | Lanza `InvalidOperationException` | Devuelve el primero |
| `FirstOrDefault` | Devuelve `default` (`null`, `0`) | Devuelve el primero |
| `Single` | Lanza excepción | **Lanza excepción** |
| `SingleOrDefault` | Devuelve `default` | **Lanza excepción** |
| `Last` / `LastOrDefault` | Como `First`, pero desde el final | — |

Usa `Single` cuando "más de uno" sería un error de datos (buscar por un id único). Desde .NET 6 puedes indicar el valor por defecto: `numeros.FirstOrDefault(n => n > 100, -1)`.

### Preguntar: `Any`, `All`, `Contains`

```csharp
bool hayDe2005 = libros.Any(l => l.Publicacion.Year == 2005);            // false
bool hayLibros = libros.Any();                                            // true si no está vacía
bool todosPublicados = libros.All(l => l.Estado == "PUBLISH");           // false (hay un MEAP)
bool todosConTitulo = libros.All(l => !string.IsNullOrEmpty(l.Titulo));  // true
bool tieneClean = libros.Select(l => l.Titulo).Contains("Clean Code");   // true
```

* `Any` se detiene en el primero que cumple; `All`, en el primero que no cumple.
* `All` sobre una secuencia **vacía** devuelve `true` (no hay ningún elemento que incumpla).
* Para saber si hay elementos, `Any()` es mejor que `Count() > 0`.

### Proyectar: `Select` y `SelectMany`

```csharp
var titulos = libros.Select(l => l.Titulo);                                   // IEnumerable<string>
var resumen = libros.Select(l => new { l.Titulo, Anio = l.Publicacion.Year });
var numerados = libros.Select((l, i) => $"{i + 1}. {l.Titulo}");             // con la posición
```

`SelectMany` **aplana**: cada libro tiene un array de categorías y quieres una sola lista de todas ellas:

```csharp
var conSelect = libros.Select(l => l.Categorias);          // IEnumerable<string[]>: una lista de arrays
var todas = libros.SelectMany(l => l.Categorias);          // IEnumerable<string>: todas en una sola secuencia
var categoriasUnicas = libros.SelectMany(l => l.Categorias).Distinct().Order();
// Java, Programación, Python
```

-----

## Ejemplo completo

Un buscador de catálogo con filtros opcionales, orden y paginación:

```csharp
var libros = new List<Libro>
{
    new("Clean Code", 464, new(2008, 8, 1), new[] { "Programación", "Java" }, "PUBLISH"),
    new("Java Concurrency", 425, new(2006, 5, 9), new[] { "Java" }, "PUBLISH"),
    new("Python Crash Course", 544, new(2019, 5, 3), new[] { "Python" }, "PUBLISH"),
    new("Effective Java", 412, new(2018, 1, 6), new[] { "Java" }, "PUBLISH"),
    new("Fluent Python", 0, new(2015, 8, 20), new[] { "Python" }, "MEAP"),
    new("Refactoring", 448, new(2018, 11, 19), new[] { "Programación" }, "PUBLISH"),
    new("Head First Java", 688, new(2005, 2, 9), new[] { "Java" }, "PUBLISH"),
};

var filtro = new FiltroBusqueda(Categoria: "Java", MinPaginas: 400, Pagina: 1, TamanoPagina: 3);
var resultado = Buscar(libros, filtro);

Console.WriteLine($"Página {filtro.Pagina} de {resultado.TotalPaginas} ({resultado.Total} resultados):");
foreach (var titulo in resultado.Items) Console.WriteLine($"  {titulo}");

Console.WriteLine($"¿Algún libro de 2005? {libros.Any(l => l.Publicacion.Year == 2005)}");
Console.WriteLine($"¿Todos publicados? {libros.All(l => l.Estado == "PUBLISH")}");
Console.WriteLine($"Primer libro de Python: {libros.FirstOrDefault(l => l.Categorias.Contains("Python"))?.Titulo ?? "ninguno"}");
Console.WriteLine($"Categorías: {string.Join(", ", libros.SelectMany(l => l.Categorias).Distinct().Order())}");

static Pagina<string> Buscar(IEnumerable<Libro> libros, FiltroBusqueda f)
{
    IEnumerable<Libro> consulta = libros;

    if (f.Categoria is not null)
        consulta = consulta.Where(l => l.Categorias.Contains(f.Categoria));   // los filtros se agregan
    if (f.MinPaginas > 0)
        consulta = consulta.Where(l => l.Paginas >= f.MinPaginas);           // solo si hacen falta

    var ordenada = consulta
        .OrderByDescending(l => l.Publicacion)
        .ThenBy(l => l.Titulo)
        .ToList();                                     // materializo una vez: la uso para contar y paginar

    var items = ordenada
        .Skip((f.Pagina - 1) * f.TamanoPagina)
        .Take(f.TamanoPagina)
        .Select(l => $"{l.Titulo} ({l.Publicacion.Year}, {l.Paginas} págs.)")
        .ToList();

    int totalPaginas = (int)Math.Ceiling(ordenada.Count / (double)f.TamanoPagina);
    return new Pagina<string>(items, ordenada.Count, totalPaginas);
}

record Libro(string Titulo, int Paginas, DateTime Publicacion, string[] Categorias, string Estado);
record FiltroBusqueda(string? Categoria, int MinPaginas, int Pagina, int TamanoPagina);
record Pagina<T>(List<T> Items, int Total, int TotalPaginas);
```

Salida:

```text
Página 1 de 2 (4 resultados):
  Effective Java (2018, 412 págs.)
  Clean Code (2008, 464 págs.)
  Java Concurrency (2006, 425 págs.)
¿Algún libro de 2005? True
¿Todos publicados? False
Primer libro de Python: Python Crash Course
Categorías: Java, Programación, Python
```

Fíjate en cómo se construye la consulta **por partes**: cada `Where` se agrega solo si el filtro está presente. Como la ejecución es diferida, nada corre hasta el `ToList()`. Es el patrón habitual para búsquedas con filtros opcionales, también con Entity Framework.

-----

## Errores comunes

**1. `First` sobre una secuencia sin coincidencias.**
Qué pasa: `System.InvalidOperationException: Sequence contains no matching element`.
Por qué: `First` exige al menos un elemento que cumpla.
Arreglo: `FirstOrDefault` y comprobar `null`.

**2. `Single` cuando puede haber varios.**
Qué pasa: `InvalidOperationException: Sequence contains more than one matching element`.
Por qué: `Single` exige exactamente uno.
Arreglo: `First` si cualquiera sirve; `Single` solo cuando "más de uno" es un error de datos.

**3. Dos `OrderBy` seguidos.**
Qué pasa: el resultado queda ordenado solo por el segundo criterio.
Por qué: cada `OrderBy` empieza un ordenamiento nuevo.
Arreglo: `OrderBy(...).ThenBy(...)`.

**4. Paginar sin ordenar.**
Qué pasa: los elementos aparecen repetidos o se pierden entre páginas (sobre todo con bases de datos).
Por qué: sin un orden definido, el origen puede devolver los datos en cualquier orden en cada consulta.
Arreglo: siempre `OrderBy` antes de `Skip`/`Take`, idealmente con un desempate único (id).

**5. Confundir `TakeWhile` con `Where`.**
Qué pasa: faltan resultados.
Por qué: `TakeWhile` corta en el primer elemento que no cumple.
Arreglo: usa `Where` si quieres revisar todos.

**6. Usar `Select` cuando querías aplanar.**
Qué pasa: obtienes `IEnumerable<string[]>` y el `foreach` imprime `System.String[]`.
Por qué: `Select` mantiene la estructura anidada.
Arreglo: `SelectMany`.

-----

## Según la versión de C#

* **.NET Framework 3.5:** los operadores básicos (`Where`, `OrderBy`, `Skip`, `Take`, `First`, `Any`...).
* **.NET Core 2.0:** `TakeLast` y `SkipLast`.
* **.NET 6:** `DistinctBy`, `Chunk`, `Take(rango)` (como `Take(2..5)`) y valores por defecto en `FirstOrDefault`/`SingleOrDefault`/`LastOrDefault`.
* **.NET 7:** `Order` y `OrderDescending`.
* **.NET 9:** `Index()`, que devuelve pares `(índice, elemento)`.

-----

## Cuándo sí y cuándo no

**Usa estos operadores cuando:**

* Lees o presentas datos: filtros de búsqueda, listados ordenados, páginas, validaciones (`All`, `Any`).

**Ten en cuenta:**

* `OrderBy` necesita leer **toda** la secuencia antes de devolver el primer elemento. Sobre una secuencia infinita, no termina nunca.
* `Count()`, `Last()` y `ElementAt()` pueden recorrer toda la secuencia si no es una colección con índice.
* En listas pequeñas y código crítico, un bucle puede ser más rápido; en el resto de los casos, prioriza la legibilidad.

-----

## Resumen en 5 líneas

1. Filtrar: `Where`, `Distinct`/`DistinctBy`, `OfType<T>`. Ordenar: `OrderBy` + `ThenBy` (no dos `OrderBy`).
2. Paginar: ordenar, luego `Skip((página - 1) * tamaño).Take(tamaño)`; el orden de `Skip`/`Take` importa.
3. `First`/`Single` lanzan excepción si no hay coincidencias; las versiones `OrDefault` devuelven `default`.
4. `Any` (¿alguno?), `All` (¿todos?, `true` si está vacía) y `Contains` responden sí o no.
5. `Select` transforma uno a uno; `SelectMany` aplana colecciones anidadas.

-----

## Para profundizar

<details>
<summary>Ordenar con comparadores y claves compuestas</summary>

```csharp
var porNombreSinMayusculas = nombres.OrderBy(n => n, StringComparer.OrdinalIgnoreCase);

var porVarias = libros.OrderBy(l => (l.Categorias[0], -l.Paginas));   // tupla como clave
```

Las tuplas se comparan elemento por elemento, así que una tupla como clave ordena por varios criterios a la vez (aunque `ThenBy` suele ser más legible).

</details>

<details>
<summary>Paginación por "cursor" (keyset)</summary>

`Skip(n)` con bases de datos obliga a leer y descartar `n` filas: con páginas lejanas se vuelve lento. La alternativa es paginar por la última clave vista:

```csharp
var siguientePagina = libros
    .Where(l => l.Publicacion < ultimaFechaVista)
    .OrderByDescending(l => l.Publicacion)
    .Take(10);
```

Es el patrón de los "cargar más" y de las APIs con `nextCursor`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`Where` filtra, `OrderBy` y `ThenBy` ordenan, `Skip` y `Take` sirven para paginar, `First` y `Single` obtienen un elemento (y lanzan una excepción si no hay), `FirstOrDefault` devuelve `null` si no encuentra, y `Any` y `All` responden si alguno o todos cumplen una condición. `Select` transforma y `SelectMany` aplana listas anidadas.

### Respuesta ampliada (semi-senior)

`OrderBy` es estable y diferido, pero bloqueante: consume toda la fuente en el primer `MoveNext`; los criterios secundarios van en `ThenBy`, porque un nuevo `OrderBy` descarta el anterior. `Skip`/`Take` deben aplicarse sobre una secuencia con orden determinista (en SQL, con un desempate único), y para páginas profundas conviene la paginación por cursor. `First` frente a `Single` expresa una intención: `Single` valida la unicidad y es más costoso porque necesita comprobar que no haya un segundo. `Any()` corta en el primer elemento y es preferible a `Count() > 0`. `SelectMany` es el *flatMap* de LINQ; `Chunk`, `DistinctBy` y compañía (.NET 6) reemplazan muchos patrones manuales.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `First` y `Single`?**
`First` devuelve el primero que cumple, aunque haya más. `Single` exige que haya exactamente uno; si hay más, lanza una excepción.

**2. ¿Diferencia entre `Select` y `SelectMany`?**
`Select` produce un resultado por elemento (puede ser una colección, y quedan anidadas). `SelectMany` aplana esas colecciones en una sola secuencia.

**3. ¿Cómo paginarías una consulta?**
`OrderBy` con un criterio determinista, luego `Skip((pagina - 1) * tamano).Take(tamano)`, y aparte el total con `Count()` para calcular las páginas.

-----

## Práctica

**Ejercicio 1.** Con la lista `libros` de la lección, obtén los títulos de los libros publicados a partir de 2010, ordenados por cantidad de páginas de mayor a menor y, ante empate, por título.

<details>
<summary>Solución</summary>

```csharp
var titulos = libros
    .Where(l => l.Publicacion.Year >= 2010)
    .OrderByDescending(l => l.Paginas)
    .ThenBy(l => l.Titulo)
    .Select(l => l.Titulo);

Console.WriteLine(string.Join(" | ", titulos));
// Python Crash Course | Refactoring | Effective Java | Fluent Python
```

</details>

**Ejercicio 2.** Dado `int[] temperaturas = { 18, 21, 25, 31, 28, 22, 35, 19 };`, obtén: (a) las temperaturas hasta la primera que supera 30, (b) si alguna supera 34, (c) la primera mayor a 40 o `-1` si no hay, (d) la página 2 de tamaño 3.

<details>
<summary>Solución</summary>

```csharp
int[] temperaturas = { 18, 21, 25, 31, 28, 22, 35, 19 };

var a = temperaturas.TakeWhile(t => t <= 30);                    // 18, 21, 25
bool b = temperaturas.Any(t => t > 34);                          // True
int c = temperaturas.FirstOrDefault(t => t > 40, -1);            // -1
var d = temperaturas.Skip(3).Take(3);                            // 31, 28, 22

Console.WriteLine($"{string.Join(",", a)} | {b} | {c} | {string.Join(",", d)}");
```

</details>

-----

## Siguiente lección

[Agregar, agrupar y unir](03-Agregar%20agrupar%20y%20unir.md)
