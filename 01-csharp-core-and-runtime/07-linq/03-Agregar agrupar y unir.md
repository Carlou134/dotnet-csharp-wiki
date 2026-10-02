# Agregar, agrupar y unir

## En una frase

LINQ resume secuencias en un valor (`Count`, `Sum`, `Average`, `Min`/`Max`, `MinBy`/`MaxBy`, `Aggregate`), las **agrupa** por una clave (`GroupBy`, `ToLookup`), las **combina** con otras (`Join`, `GroupJoin`, `Zip`) y hace operaciones de conjuntos entre secuencias (`Union`, `Intersect`, `Except`).

-----

## Antes de empezar

Conviene que ya sepas:

* Filtrar, ordenar, proyectar y la diferencia entre operadores diferidos e inmediatos, de [Filtrar, ordenar y paginar](02-Filtrar%20ordenar%20y%20paginar.md).
* Diccionarios y conjuntos, de [Diccionarios y conjuntos](../06-colecciones/02-Diccionarios%20y%20conjuntos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Agregación:** reducir una secuencia a un solo valor (total, promedio, máximo).
* **Acumulador:** el valor que se va construyendo en `Aggregate`.
* **Agrupamiento:** reunir elementos que comparten una clave.
* **`IGrouping<TKey, TElement>`:** un grupo: una clave (`Key`) y los elementos que la comparten.
* **Lookup:** estructura tipo diccionario donde cada clave tiene **varios** valores.
* **Join (unión):** combinar dos secuencias emparejando elementos con una clave común.
* **Formato compuesto:** `{0,-20}`, alinear columnas al imprimir.

-----

## El problema

Un reporte de ventas típico pide:

* Total vendido, promedio por venta, venta más grande.
* Ventas **por categoría**: cantidad y total de cada una.
* Nombre del **cliente** de cada venta, que está en **otra** lista (las ventas solo guardan el id del cliente).

Con bucles: un diccionario para acumular por categoría, variables para el máximo, un bucle anidado (o un diccionario auxiliar) para buscar cada cliente... Fácilmente 40 líneas, con varios lugares donde equivocarse. Con LINQ, cada pregunta del reporte es una expresión.

-----

## Cómo funciona

Los ejemplos usan estos datos:

```csharp
record Venta(int Id, int ClienteId, string Categoria, decimal Monto, DateTime Fecha);
record Cliente(int Id, string Nombre, string Ciudad);

var ventas = new List<Venta>
{
    new(1, 10, "Libros", 120m, new(2026, 1, 5)),
    new(2, 11, "Música", 45m, new(2026, 1, 9)),
    new(3, 10, "Libros", 80m, new(2026, 2, 1)),
    new(4, 12, "Juegos", 300m, new(2026, 2, 14)),
    new(5, 11, "Libros", 60m, new(2026, 3, 2)),
};

var clientes = new List<Cliente>
{
    new(10, "Ana", "Lima"),
    new(11, "Luis", "Cusco"),
    new(12, "Eva", "Lima"),
    new(13, "Juan", "Piura"),   // no tiene ventas
};
```

### Agregaciones

```csharp
int cantidad = ventas.Count();                            // 5
int deLibros = ventas.Count(v => v.Categoria == "Libros");  // 3 (con filtro)
long muchas = ventas.LongCount();                        // igual, pero devuelve long

decimal total = ventas.Sum(v => v.Monto);                // 605
decimal promedio = ventas.Average(v => v.Monto);         // 121
decimal maximo = ventas.Max(v => v.Monto);               // 300 (el VALOR)
DateTime primera = ventas.Min(v => v.Fecha);             // 2026-01-05
```

`Max` y `Min` devuelven el **valor** máximo. Si quieres el **elemento** que lo tiene, usa `MaxBy` y `MinBy` (.NET 6):

```csharp
Venta mayor = ventas.MaxBy(v => v.Monto)!;               // la venta 4 completa
Venta masVieja = ventas.MinBy(v => v.Fecha)!;            // la venta 1 completa
```

Con secuencias **vacías**:

* `Count` y `Sum` devuelven `0`.
* `Average`, `Max` y `Min` (de tipos de valor) lanzan `InvalidOperationException: Sequence contains no elements`.
* Para evitarlo: `DefaultIfEmpty()` o tipos anulables (`ventas.Max(v => (decimal?)v.Monto)` devuelve `null`).

### `Aggregate`: tu propia reducción

```csharp
int[] numeros = { 1, 2, 3, 4 };

int producto = numeros.Aggregate((acc, n) => acc * n);                 // 24
string lista = numeros.Aggregate("", (acc, n) => acc == "" ? $"{n}" : $"{acc}-{n}");   // "1-2-3-4"
```

`Aggregate` recibe un valor inicial (opcional) y una función `(acumulador, siguiente) => nuevoAcumulador` que se aplica a cada elemento. Es poderosa pero suele ser la opción menos legible: para unir textos, `string.Join("-", numeros)` es más clara; para sumar, `Sum`.

### `GroupBy`: agrupar por una clave

```csharp
var porCategoria = ventas.GroupBy(v => v.Categoria);

foreach (IGrouping<string, Venta> grupo in porCategoria)
{
    Console.WriteLine($"{grupo.Key}: {grupo.Count()} ventas, total {grupo.Sum(v => v.Monto)}");
}
// Libros: 3 ventas, total 260
// Música: 1 ventas, total 45
// Juegos: 1 ventas, total 300
```

Cada grupo es un `IGrouping<TKey, TElement>`: tiene una `Key` y, al mismo tiempo, **es** una secuencia con los elementos de ese grupo, sobre la que puedes usar cualquier operador de LINQ.

Lo habitual es proyectar cada grupo a un resumen:

```csharp
var resumen = ventas
    .GroupBy(v => v.Categoria)
    .Select(g => new { Categoria = g.Key, Cantidad = g.Count(), Total = g.Sum(v => v.Monto) })
    .OrderByDescending(r => r.Total);
```

Agrupar por varios campos con una tupla o un tipo anónimo:

```csharp
var porMesYCategoria = ventas.GroupBy(v => (v.Fecha.Month, v.Categoria));
```

En sintaxis de consulta:

```csharp
var grupos =
    from v in ventas
    group v by v.Categoria into g
    select new { g.Key, Total = g.Sum(x => x.Monto) };
```

Para contar por clave, .NET 9 trae un atajo: `ventas.CountBy(v => v.Categoria)`.

### `ToLookup`: un diccionario con varios valores por clave

```csharp
ILookup<char, Cliente> porInicial = clientes.ToLookup(c => c.Nombre[0]);

foreach (var c in porInicial['L']) Console.WriteLine(c.Nombre);   // Luis
Console.WriteLine(porInicial['Z'].Count());                        // 0: no lanza excepción
```

| | `GroupBy` | `ToLookup` |
| --- | --- | --- |
| Ejecución | Diferida | **Inmediata** (materializa) |
| Acceso por clave | No (se recorre) | Sí: `lookup[clave]` |
| Clave inexistente | — | Devuelve una secuencia vacía |

`ToDictionary` exige claves únicas (lanza excepción con duplicados); `ToLookup` admite varias entradas por clave.

### `Join`: combinar dos secuencias por una clave

Las ventas tienen `ClienteId`; los nombres están en `clientes`. `Join` empareja los elementos cuya clave coincide:

```csharp
var ventasConCliente = ventas.Join(
    clientes,                          // la otra secuencia
    v => v.ClienteId,                  // clave en ventas
    c => c.Id,                         // clave en clientes
    (v, c) => new { v.Id, c.Nombre, v.Monto });   // qué devolver por cada par

foreach (var x in ventasConCliente)
    Console.WriteLine($"Venta {x.Id}: {x.Nombre} - {x.Monto}");
```

En sintaxis de consulta (aquí es donde más se lee):

```csharp
var mismo =
    from v in ventas
    join c in clientes on v.ClienteId equals c.Id
    select new { v.Id, c.Nombre, v.Monto };
```

`Join` es un **inner join**: los elementos sin pareja (Juan, que no tiene ventas) no aparecen.

### `GroupJoin` y left join

`GroupJoin` empareja cada elemento con **todos** los de la otra secuencia que coinciden, incluso si son cero:

```csharp
var clientesConVentas = clientes.GroupJoin(
    ventas,
    c => c.Id,
    v => v.ClienteId,
    (c, susVentas) => new { c.Nombre, Total = susVentas.Sum(v => v.Monto) });

// Ana 200, Luis 105, Eva 300, Juan 0  ← Juan aparece con 0
```

Es la forma de hacer un *left join*: todos los clientes, tengan o no ventas. (.NET 10 agrega `LeftJoin` y `RightJoin` como operadores directos).

### Combinar secuencias: `Zip`, `Concat` y operaciones de conjuntos

```csharp
string[] nombres = { "Ana", "Luis", "Eva" };
int[] puntajes = { 90, 75, 88 };
var pares = nombres.Zip(puntajes, (n, p) => $"{n}: {p}");   // "Ana: 90", "Luis: 75", "Eva: 88"

int[] a = { 1, 2, 3, 4 };
int[] b = { 3, 4, 5 };
var todos = a.Concat(b);       // 1, 2, 3, 4, 3, 4, 5 (con repetidos)
var union = a.Union(b);        // 1, 2, 3, 4, 5
var comunes = a.Intersect(b);  // 3, 4
var soloA = a.Except(b);       // 1, 2
```

A diferencia de los métodos de `HashSet<T>`, estos operadores **no modifican** nada: devuelven secuencias nuevas. Existen también versiones con clave: `UnionBy`, `IntersectBy`, `ExceptBy` (.NET 6).

-----

## Ejemplo completo

Reporte de ventas con formato de columnas:

```csharp
var ventas = new List<Venta>
{
    new(1, 10, "Libros", 120m, new(2026, 1, 5)),
    new(2, 11, "Música", 45m, new(2026, 1, 9)),
    new(3, 10, "Libros", 80m, new(2026, 2, 1)),
    new(4, 12, "Juegos", 300m, new(2026, 2, 14)),
    new(5, 11, "Libros", 60m, new(2026, 3, 2)),
};

var clientes = new List<Cliente>
{
    new(10, "Ana", "Lima"),
    new(11, "Luis", "Cusco"),
    new(12, "Eva", "Lima"),
    new(13, "Juan", "Piura"),
};

// 1. Totales generales
Console.WriteLine($"Ventas: {ventas.Count}  Total: {ventas.Sum(v => v.Monto):N2}  Promedio: {ventas.Average(v => v.Monto):N2}");
var mayor = ventas.MaxBy(v => v.Monto)!;
Console.WriteLine($"Venta más grande: #{mayor.Id} ({mayor.Monto:N2})");
Console.WriteLine();

// 2. Por categoría (formato compuesto para alinear columnas)
Console.WriteLine("{0,-10} {1,8} {2,10}", "Categoría", "Ventas", "Total");
foreach (var g in ventas.GroupBy(v => v.Categoria).OrderByDescending(g => g.Sum(v => v.Monto)))
{
    Console.WriteLine("{0,-10} {1,8} {2,10:N2}", g.Key, g.Count(), g.Sum(v => v.Monto));
}
Console.WriteLine();

// 3. Total por cliente, incluidos los que no compraron (left join)
var porCliente =
    from c in clientes
    join v in ventas on c.Id equals v.ClienteId into susVentas
    select new { c.Nombre, c.Ciudad, Total = susVentas.Sum(x => x.Monto) };

foreach (var x in porCliente.OrderByDescending(x => x.Total))
{
    Console.WriteLine($"{x.Nombre,-6} {x.Ciudad,-6} {x.Total,8:N2}");
}
Console.WriteLine();

// 4. Total por ciudad: join + group
var porCiudad = ventas
    .Join(clientes, v => v.ClienteId, c => c.Id, (v, c) => new { c.Ciudad, v.Monto })
    .GroupBy(x => x.Ciudad)
    .Select(g => $"{g.Key}: {g.Sum(x => x.Monto):N2}");
Console.WriteLine(string.Join(" | ", porCiudad));

record Venta(int Id, int ClienteId, string Categoria, decimal Monto, DateTime Fecha);
record Cliente(int Id, string Nombre, string Ciudad);
```

Salida:

```text
Ventas: 5  Total: 605.00  Promedio: 121.00
Venta más grande: #4 (300.00)

Categoría    Ventas      Total
Juegos            1     300.00
Libros            3     260.00
Música            1      45.00

Eva    Lima     300.00
Ana    Lima     200.00
Luis   Cusco    105.00
Juan   Piura      0.00

Lima: 500.00 | Cusco: 105.00
```

`join ... into` en la sintaxis de consulta es un `GroupJoin`: por eso Juan aparece con 0.

-----

## Errores comunes

**1. `Average`, `Max` o `Min` sobre una secuencia vacía.**
Qué pasa: `System.InvalidOperationException: Sequence contains no elements`.
Por qué: no hay un promedio o máximo de "nada".
Arreglo: comprueba con `Any()` o usa un tipo anulable: `Max(v => (decimal?)v.Monto) ?? 0`.

**2. Usar `Max` cuando querías el elemento.**
Qué pasa: obtienes `300` y luego buscas la venta con un segundo recorrido (`First(v => v.Monto == 300)`).
Por qué: `Max` devuelve el valor.
Arreglo: `MaxBy(v => v.Monto)`.

**3. `ToDictionary` con claves repetidas.**
Qué pasa: `System.ArgumentException: An item with the same key has already been added.`
Por qué: un diccionario no admite claves duplicadas.
Arreglo: `GroupBy`/`ToLookup`, o `DistinctBy` antes si de verdad quieres uno por clave.

**4. Esperar los elementos sin pareja en un `Join`.**
Qué pasa: los clientes sin ventas "desaparecen" del reporte.
Por qué: `Join` es un inner join.
Arreglo: `GroupJoin` (o `join ... into`) para un left join.

**5. Usar `Aggregate` para todo.**
Qué pasa: código difícil de leer y, con strings, lento (concatena en cada paso).
Por qué: `Aggregate` es la herramienta más general, no la más clara.
Arreglo: `Sum`, `Count`, `string.Join`, y deja `Aggregate` para reducciones que no tienen operador propio.

**6. Recorrer un `GroupBy` varias veces.**
Qué pasa: se reagrupa todo en cada recorrido.
Por qué: `GroupBy` es diferido.
Arreglo: materializa (`ToList()`) o usa `ToLookup` si vas a consultar varias veces.

-----

## Según la versión de C#

* **.NET Framework 3.5:** agregaciones, `GroupBy`, `ToLookup`, `Join`, `GroupJoin` y operaciones de conjuntos.
* **.NET 4.0:** `Zip`.
* **.NET 6:** `MinBy`, `MaxBy`, `UnionBy`, `IntersectBy`, `ExceptBy` y `Zip` de tres secuencias.
* **.NET 9:** `CountBy` y `AggregateBy`, que agrupan y resumen en un solo paso.
* **.NET 10:** `LeftJoin` y `RightJoin`.

-----

## Cuándo sí y cuándo no

**Usa agregaciones y agrupamientos cuando:**

* Construyes reportes, estadísticas o resúmenes a partir de colecciones en memoria.

**Usa `Join` cuando:**

* Combinas dos colecciones en memoria relacionadas por una clave. Si haces muchos joins sobre los mismos datos, un `Dictionary` por id suele ser más simple y rápido.

**Ten en cuenta:**

* Con bases de datos (EF Core), estos operadores se traducen a SQL (`GROUP BY`, `JOIN`, `SUM`): es mejor agregar en la base de datos que traer todas las filas a memoria.

-----

## Resumen en 5 líneas

1. `Count`, `Sum`, `Average`, `Min`, `Max` resumen; `MinBy`/`MaxBy` devuelven el elemento; `Aggregate` permite reducciones propias.
2. `Average`/`Max`/`Min` lanzan excepción con secuencias vacías; `Count` y `Sum` devuelven 0.
3. `GroupBy` crea grupos (`Key` + elementos); `ToLookup` lo mismo pero inmediato y accesible por clave.
4. `Join` es un inner join por clave; `GroupJoin` (o `join ... into`) permite un left join.
5. `Zip` combina por posición; `Union`, `Intersect`, `Except` hacen operaciones de conjuntos sin modificar nada.

-----

## Para profundizar

<details>
<summary>Formato compuesto para tablas en consola</summary>

En `Console.WriteLine("{0,-10} {1,8} {2,10:N2}", a, b, c)`:

* `{0,-10}`: primer argumento, alineado a la **izquierda** en 10 caracteres.
* `{1,8}`: segundo argumento, alineado a la **derecha** en 8 caracteres.
* `{2,10:N2}`: tercero, a la derecha en 10 caracteres y con formato numérico de 2 decimales.

La interpolación admite lo mismo: `$"{nombre,-10}{total,10:N2}"`.

</details>

<details>
<summary>AggregateBy y CountBy (.NET 9)</summary>

```csharp
var totales = ventas.AggregateBy(
    v => v.Categoria,          // clave
    0m,                        // valor inicial por grupo
    (acc, v) => acc + v.Monto);  // acumulación

foreach (var (categoria, total) in totales)
    Console.WriteLine($"{categoria}: {total}");
```

Calcula el resultado por clave sin crear los grupos intermedios de `GroupBy`, lo que ahorra memoria con grandes volúmenes.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para resumir datos uso `Count`, `Sum`, `Average`, `Min` y `Max`; para agrupar, `GroupBy`, que devuelve grupos con una `Key` y sus elementos. `Join` combina dos colecciones por una clave común, como un `INNER JOIN` de SQL, y `GroupJoin` permite incluir los elementos que no tienen pareja.

### Respuesta ampliada (semi-senior)

Las agregaciones son operadores inmediatos; `Average`/`Min`/`Max` sobre tipos de valor lanzan excepción con secuencias vacías, a diferencia de sus sobrecargas anulables. `MinBy`/`MaxBy` evitan el doble recorrido para obtener el elemento. `GroupBy` es diferido pero consume toda la fuente al primer acceso, y `ToLookup` es su versión materializada e indexable. `Join` construye internamente un lookup de la segunda secuencia (O(n + m)) y es un inner join; el left join se expresa con `GroupJoin` + `SelectMany`/`DefaultIfEmpty` o, desde .NET 10, con `LeftJoin`. Con `IQueryable` estos operadores se traducen a SQL, por lo que conviene agregar en el servidor.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `Max` y `MaxBy`?**
`Max` devuelve el valor máximo de una propiedad; `MaxBy` devuelve el elemento que tiene ese valor.

**2. ¿Diferencia entre `GroupBy` y `ToLookup`?**
`GroupBy` es diferido y se recorre; `ToLookup` se ejecuta de inmediato y permite acceder a un grupo por clave, devolviendo una secuencia vacía si no existe.

**3. ¿Cómo harías un left join en LINQ?**
Con `GroupJoin` (o `join ... into` en sintaxis de consulta), y opcionalmente `SelectMany` con `DefaultIfEmpty` para aplanarlo.

-----

## Práctica

**Ejercicio 1.** Con `ventas` de la lección, obtén por cada mes (número) la cantidad de ventas y el monto promedio, ordenado por mes.

<details>
<summary>Solución</summary>

```csharp
var porMes = ventas
    .GroupBy(v => v.Fecha.Month)
    .OrderBy(g => g.Key)
    .Select(g => $"Mes {g.Key}: {g.Count()} ventas, promedio {g.Average(v => v.Monto):N2}");

foreach (var linea in porMes) Console.WriteLine(linea);
// Mes 1: 2 ventas, promedio 82.50
// Mes 2: 2 ventas, promedio 190.00
// Mes 3: 1 ventas, promedio 60.00
```

</details>

**Ejercicio 2.** Usando `ventas` y `clientes`, muestra el nombre del cliente que más gastó en total y cuánto gastó.

<details>
<summary>Solución</summary>

```csharp
var top = ventas
    .GroupBy(v => v.ClienteId)
    .Select(g => new { ClienteId = g.Key, Total = g.Sum(v => v.Monto) })
    .MaxBy(x => x.Total)!;

string nombre = clientes.First(c => c.Id == top.ClienteId).Nombre;
Console.WriteLine($"{nombre}: {top.Total:N2}");   // Eva: 300.00
```

</details>

-----

## Siguiente lección

Terminaste el módulo de LINQ. Continúa con [Excepciones](../08-excepciones/README.md).
