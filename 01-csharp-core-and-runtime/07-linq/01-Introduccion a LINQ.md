# Introducción a LINQ

## En una frase

LINQ (*Language-Integrated Query*) es un conjunto de operadores (`Where`, `Select`, `OrderBy`...) que permite **consultar y transformar cualquier colección** de forma declarativa, ya sea con **sintaxis de método** (`numeros.Where(n => n > 5)`) o con **sintaxis de consulta** (`from n in numeros where n > 5 select n`), y cuyas consultas se ejecutan de forma **diferida**.

-----

## Antes de empezar

Conviene que ya sepas:

* Lambdas y `Func<T, TResult>`, de [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).
* `IEnumerable<T>`, iteradores y ejecución diferida, de [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md).
* La diferencia entre estilo imperativo y declarativo, de [Paradigmas de programación](../00-introduccion/04-Paradigmas%20de%20programacion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **LINQ:** *Language-Integrated Query*, consultas integradas en el lenguaje.
* **Operador de consulta:** cada operación de LINQ (`Where`, `Select`, `OrderBy`...).
* **Sintaxis de método:** encadenar llamadas: `.Where(...).Select(...)`.
* **Sintaxis de consulta:** palabras clave parecidas a SQL: `from ... where ... select`.
* **Proyección:** transformar cada elemento en otra cosa (`Select`).
* **Tipo anónimo:** objeto sin clase declarada: `new { Nombre, Edad }`.
* **Materializar:** ejecutar la consulta y guardar el resultado en una colección (`ToList()`, `ToArray()`).
* **Método de extensión:** método estático que se usa como si fuera de instancia; LINQ está hecho de métodos de extensión sobre `IEnumerable<T>`.

-----

## El problema

Tienes una lista de personajes de un juego y quieres los nombres, en mayúsculas, de los que tienen nivel mayor a 10, ordenados alfabéticamente. De forma imperativa:

```csharp
var resultado = new List<string>();
foreach (var p in personajes)
{
    if (p.Nivel > 10)
    {
        resultado.Add(p.Nombre.ToUpper());
    }
}
resultado.Sort();
```

Funciona, pero tienes que leer seis líneas para entender **qué** se quería; el **cómo** (la lista temporal, el `if`, el `Add`, el `Sort`) tapa la intención. Y para cada nueva consulta (los de nivel bajo, el promedio, agrupar por clase) escribes otro bucle parecido.

Con LINQ describes **qué** quieres:

```csharp
var resultado = personajes
    .Where(p => p.Nivel > 10)
    .Select(p => p.Nombre.ToUpper())
    .OrderBy(nombre => nombre);
```

-----

## Cómo funciona

### Qué es (y qué no es) LINQ

* Apareció en **C# 3 / .NET 3.5 (2007)**.
* **No** es un lenguaje aparte, ni parte de SQL, ni una librería de terceros: es parte de .NET (espacio de nombres `System.Linq`, importado por defecto con `ImplicitUsings`).
* Son **métodos de extensión** sobre `IEnumerable<T>`: funciona con arrays, listas, diccionarios, conjuntos, strings, resultados de iteradores... con cualquier cosa que se pueda recorrer con `foreach`.
* El mismo estilo sirve para otras fuentes: bases de datos (Entity Framework), XML (LINQ to XML).

### Sintaxis de método

```csharp
string[] heroes = { "D. Va", "Lucio", "Mercy", "Soldier 76", "Pharah", "Reinhardt" };

var cortos = heroes.Where(h => h.Length < 8);          // filtrar
var gritando = heroes.Select(h => h.ToUpper());        // transformar

var largosGritando = heroes                            // encadenar
    .Where(h => h.Length > 6)
    .Select(h => h.ToUpper());

foreach (var h in largosGritando) Console.WriteLine(h);   // SOLDIER 76, REINHARDT
```

* `Where(predicado)`: deja pasar los elementos para los que la lambda devuelve `true`.
* `Select(transformacion)`: convierte cada elemento en otro valor.
* Cada operador devuelve una **nueva secuencia**; la colección original no cambia.

Piensa en el encadenamiento como una cinta transportadora: cada línea es una estación que filtra o transforma lo que pasa por ella. Y los elementos viajan **de a uno** por toda la cadena:

```text
heroes          Where(h.Length > 6)        Select(ToUpper)          foreach
"D. Va"   ──►   ✘ descartado
"Lucio"   ──►   ✘ descartado
"Soldier 76" ►  ✔ pasa   ──────────────►   "SOLDIER 76"   ──────►   imprime
"Reinhardt" ─►  ✔ pasa   ──────────────►   "REINHARDT"    ──────►   imprime
```

No se crea una lista intermedia con los filtrados: cada elemento atraviesa todas las estaciones antes de que se lea el siguiente.

### Sintaxis de consulta

```csharp
var largosGritando2 =
    from h in heroes
    where h.Length > 6
    select h.ToUpper();
```

* `from h in heroes`: declara la variable de recorrido.
* `where`: filtra (opcional).
* `select`: indica qué se devuelve por cada elemento (obligatorio, o `group ... by`).

El compilador **traduce** la sintaxis de consulta a llamadas de método: las dos versiones son idénticas en ejecución.

| | Sintaxis de método | Sintaxis de consulta |
| --- | --- | --- |
| Estilo | C# encadenado | Parecido a SQL |
| Operadores disponibles | **Todos** | Solo algunos (`where`, `select`, `orderby`, `group`, `join`, `let`) |
| Uso en la práctica | La más común | Útil en consultas con `join`, `let` o varios `from` |

Puedes mezclarlas: `(from h in heroes where h.Length > 6 select h).Count()`.

### `var` y tipos anónimos

El resultado de `Where` o `Select` es un `IEnumerable<T>` cuyo tipo exacto suele ser largo o imposible de escribir, por eso se usa `var`:

```csharp
var resumen = heroes.Select(h => new { Nombre = h, Largo = h.Length });

foreach (var r in resumen)
{
    Console.WriteLine($"{r.Nombre}: {r.Largo}");
}
```

`new { Nombre = h, Largo = h.Length }` crea un **tipo anónimo**: el compilador genera una clase con esas propiedades (de solo lectura). No puedes escribir su nombre, así que `var` es obligatorio. Si el nombre de la propiedad coincide con el de la variable, puedes abreviar: `new { h.Length, p.Nombre }`.

Los tipos anónimos sirven dentro de un método. Para devolver el resultado o pasarlo a otro método, usa un `record` o una tupla.

### Ejecución diferida

Una consulta LINQ **no se ejecuta cuando la escribes**, sino cuando la recorres (igual que un iterador, porque `Where` y `Select` **son** iteradores):

```csharp
var numeros = new List<int> { 1, 2, 3 };
var mayores = numeros.Where(n => n > 1);   // todavía no filtró nada

numeros.Add(10);                            // modifico la fuente DESPUÉS de definir la consulta

foreach (int n in mayores)                  // ahora sí se ejecuta, con la lista actual
    Console.Write($"{n} ");                 // 2 3 10
```

Y se ejecuta **de nuevo** en cada recorrido:

```csharp
var caros = productos.Where(p => CalcularPrecioDesdeApi(p) > 100);   // supón que esto es lento

int cuantos = caros.Count();     // ejecuta la consulta
var lista = caros.ToList();      // ejecuta la consulta OTRA VEZ
```

### Materializar: `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`

Para ejecutar la consulta una sola vez y guardar el resultado:

```csharp
List<string> lista = heroes.Where(h => h.Length > 6).ToList();
string[] array = heroes.Select(h => h.ToUpper()).ToArray();
Dictionary<string, int> porNombre = heroes.ToDictionary(h => h, h => h.Length);
HashSet<int> largosUnicos = heroes.Select(h => h.Length).ToHashSet();
```

Regla práctica: **materializa** cuando vas a recorrer el resultado más de una vez, cuando la fuente puede cambiar o cuando quieres "congelar" el resultado en ese momento.

Algunos operadores se ejecutan **inmediatamente** porque devuelven un solo valor: `Count()`, `Sum()`, `First()`, `Any()`, `Max()`... Los que devuelven secuencias (`Where`, `Select`, `OrderBy`...) son diferidos.

### LINQ con cualquier colección

```csharp
List<string> lista = new() { "D. Va", "Lucio", "Soldier 76" };
var largos = lista.Where(h => h.Length > 6);                      // listas

var diccionario = new Dictionary<string, int> { ["a"] = 1, ["b"] = 5 };
var claves = diccionario.Where(par => par.Value > 2).Select(par => par.Key);   // diccionarios (pares)

int vocales = "programación".Count(c => "aeiouáéíóú".Contains(c));   // strings (secuencia de char)

var numeros = Enumerable.Range(1, 100).Where(n => n % 7 == 0);   // secuencias generadas
```

-----

## Ejemplo completo

```csharp
var personajes = new List<Personaje>
{
    new("Aria", "Maga", 15, 320),
    new("Bram", "Guerrero", 8, 540),
    new("Cira", "Arquera", 22, 410),
    new("Dax", "Guerrero", 12, 610),
    new("Elan", "Mago", 5, 180)
};

// 1. Sintaxis de método: filtrar, ordenar, proyectar
var veteranos = personajes
    .Where(p => p.Nivel > 10)
    .OrderBy(p => p.Nombre)
    .Select(p => $"{p.Nombre.ToUpper()} ({p.Clase}, nivel {p.Nivel})");

Console.WriteLine("Veteranos:");
foreach (var v in veteranos) Console.WriteLine($"  {v}");

// 2. Sintaxis de consulta con tipo anónimo
var poderosos =
    from p in personajes
    where p.Vida > 400
    select new { p.Nombre, PoderTotal = p.Nivel * p.Vida };

Console.WriteLine("Poderosos:");
foreach (var p in poderosos) Console.WriteLine($"  {p.Nombre}: {p.PoderTotal}");

// 3. Ejecución diferida
var guerreros = personajes.Where(p => p.Clase == "Guerrero");
Console.WriteLine($"Guerreros antes: {guerreros.Count()}");
personajes.Add(new Personaje("Faro", "Guerrero", 3, 300));
Console.WriteLine($"Guerreros después: {guerreros.Count()}");   // la consulta ve el nuevo

// 4. Materializar para congelar el resultado
var fotoDeMagos = personajes.Where(p => p.Clase.StartsWith("Mag")).ToList();
personajes.Add(new Personaje("Gala", "Maga", 9, 200));
Console.WriteLine($"Magos en la foto: {fotoDeMagos.Count}");      // no ve a Gala

record Personaje(string Nombre, string Clase, int Nivel, int Vida);
```

Salida:

```text
Veteranos:
  ARIA (Maga, nivel 15)
  CIRA (Arquera, nivel 22)
  DAX (Guerrero, nivel 12)
Poderosos:
  Bram: 4320
  Cira: 9020
  Dax: 7320
Guerreros antes: 2
Guerreros después: 3
Magos en la foto: 2
```

-----

## Errores comunes

**1. Olvidar que la consulta es diferida.**
Qué pasa: el resultado "cambia solo" porque la fuente se modificó entre la definición y el recorrido.
Por qué: la consulta se ejecuta al recorrerla, con los datos de ese momento.
Arreglo: `ToList()` si quieres el resultado de un momento concreto.

**2. Recorrer varias veces una consulta costosa.**
Qué pasa: el programa es lento o llama varias veces a una base de datos o API.
Por qué: cada `Count()`, `foreach` o `ToList()` vuelve a ejecutarla.
Arreglo: materializa una vez y reutiliza la lista.

**3. Esperar que LINQ modifique la colección.**
Qué pasa: `lista.Where(x => x > 0);` "no hace nada".
Por qué: LINQ devuelve secuencias nuevas y no modifica la fuente (y si no recorres el resultado, ni siquiera se ejecuta).
Arreglo: guarda el resultado: `lista = lista.Where(x => x > 0).ToList();` o usa `RemoveAll` si quieres modificar la lista.

**4. Usar `Count()` para saber si hay elementos.**
Qué pasa: funciona, pero recorre toda la secuencia.
Por qué: `Count()` cuenta todos (salvo que sea una colección con `Count` propio).
Arreglo: `Any()`, que se detiene en el primero.

**5. Devolver un tipo anónimo desde un método.**
Qué pasa: tienes que declarar el retorno como `object` o `dynamic` y pierdes las propiedades.
Por qué: el tipo anónimo no tiene un nombre que puedas escribir.
Arreglo: proyecta a un `record` o a una tupla con nombres.

**6. Olvidar `using System.Linq` en proyectos sin `ImplicitUsings`.**
Qué pasa: `error CS1061: 'string[]' does not contain a definition for 'Where'`.
Por qué: los métodos de extensión de LINQ están en `System.Linq`.
Arreglo: agrega `using System.Linq;` (en proyectos modernos ya viene incluido).

-----

## Según la versión de C#

* **C# 3 (2007):** LINQ, lambdas, métodos de extensión, tipos anónimos, `var` e inicializadores de objetos (todos nacieron para hacer posible LINQ).
* **C# 7:** tuplas con nombre, una alternativa a los tipos anónimos para devolver resultados.
* **.NET 6:** nuevos operadores: `DistinctBy`, `MinBy`, `MaxBy`, `Chunk`, `Take(rango)`.
* **.NET 9:** `CountBy`, `AggregateBy` e `Index`.

-----

## Cuándo sí y cuándo no

**Usa LINQ cuando:**

* Filtras, transformas, ordenas, agrupas o resumes datos: la intención queda clara en pocas líneas.

**Prefiere un bucle cuando:**

* Hay efectos secundarios en cada paso (escribir en consola, guardar en base de datos): un `foreach` lo hace explícito.
* La lógica es compleja y la cadena de LINQ se volvería ilegible.
* Es una ruta crítica de rendimiento: LINQ crea iteradores y delegados (aunque en .NET moderno muchos operadores están muy optimizados).

**Elige la sintaxis:**

* Método para la mayoría de los casos.
* Consulta cuando hay `join`, varios `from` o variables intermedias con `let`, donde se lee mejor.

-----

## Resumen en 5 líneas

1. LINQ son métodos de extensión sobre `IEnumerable<T>` para consultar cualquier colección de forma declarativa.
2. `Where` filtra y `Select` transforma; se encadenan y no modifican la colección original.
3. Sintaxis de método (`.Where(...)`) y de consulta (`from ... where ... select`) son equivalentes.
4. Las consultas son diferidas: se ejecutan al recorrerlas y de nuevo en cada recorrido.
5. `ToList()`, `ToArray()`, `ToDictionary()` materializan; `Count()`, `First()`, `Sum()` se ejecutan de inmediato.

-----

## Para profundizar

<details>
<summary>IEnumerable frente a IQueryable</summary>

LINQ sobre colecciones en memoria (LINQ to Objects) usa `IEnumerable<T>`: las lambdas se compilan a delegados y se ejecutan en C#. Entity Framework usa `IQueryable<T>`: las lambdas se compilan a **árboles de expresión**, que EF traduce a SQL y ejecuta en la base de datos.

```csharp
var caros = db.Productos.Where(p => p.Precio > 100);   // IQueryable: se traduce a WHERE Precio > 100
var lista = caros.ToList();                            // aquí viaja la consulta SQL
```

Si llamas a `ToList()` o `AsEnumerable()` antes de filtrar, traes **toda** la tabla a memoria y filtras en C#. Es uno de los errores de rendimiento más comunes con EF.

</details>

<details>
<summary>Cómo se traduce la sintaxis de consulta</summary>

```csharp
from p in personajes
where p.Nivel > 10
orderby p.Nombre
select p.Nombre
```

se convierte en:

```csharp
personajes.Where(p => p.Nivel > 10).OrderBy(p => p.Nombre).Select(p => p.Nombre)
```

La sintaxis de consulta es puro azúcar sintáctico. Cualquier tipo que tenga métodos llamados `Where`, `Select`, etc. con la forma adecuada se puede consultar con ella.

</details>

-----

## En entrevista

### Respuesta corta (junior)

LINQ es un conjunto de métodos para consultar colecciones en C#, como `Where` para filtrar o `Select` para transformar. Se puede escribir con sintaxis de método o con sintaxis de consulta, parecida a SQL. Funciona con cualquier `IEnumerable<T>` y las consultas no se ejecutan hasta que se recorren.

### Respuesta ampliada (semi-senior)

LINQ to Objects está implementado como métodos de extensión sobre `IEnumerable<T>`, en su mayoría iteradores perezosos: `Where`/`Select` son diferidos y en streaming, `OrderBy`/`GroupBy` son diferidos pero necesitan consumir toda la fuente al primer `MoveNext`, y los agregados (`Count`, `Sum`, `First`) se ejecutan de inmediato. La enumeración múltiple repite el trabajo, así que se materializa con `ToList` cuando corresponde. La sintaxis de consulta se traduce a llamadas de método. Con `IQueryable<T>` las lambdas son árboles de expresión que un proveedor (EF Core) traduce a SQL, por lo que el orden de `Where`/`ToList` define dónde se ejecuta el filtro.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es la ejecución diferida en LINQ?**
Que la consulta se define pero no se ejecuta hasta que se recorre el resultado, y se vuelve a ejecutar en cada recorrido.

**2. ¿Diferencia entre `IEnumerable<T>` e `IQueryable<T>`?**
`IEnumerable<T>` ejecuta en memoria con delegados; `IQueryable<T>` construye un árbol de expresión que un proveedor traduce (por ejemplo, a SQL) y ejecuta en el origen de datos.

**3. ¿Qué es un tipo anónimo?**
Una clase generada por el compilador con propiedades de solo lectura, creada con `new { ... }`, que solo puede usarse localmente con `var`.

-----

## Práctica

**Ejercicio 1.** Dado `int[] numeros = { 5, 12, 8, 21, 3, 16, 9 };`, escribe con sintaxis de método y con sintaxis de consulta una consulta que devuelva el doble de los números pares, y muestra el resultado.

<details>
<summary>Solución</summary>

```csharp
int[] numeros = { 5, 12, 8, 21, 3, 16, 9 };

var metodo = numeros.Where(n => n % 2 == 0).Select(n => n * 2);

var consulta =
    from n in numeros
    where n % 2 == 0
    select n * 2;

Console.WriteLine(string.Join(", ", metodo));     // 24, 16, 32
Console.WriteLine(string.Join(", ", consulta));   // 24, 16, 32
```

</details>

**Ejercicio 2.** ¿Qué imprime este código y por qué?

```csharp
var lista = new List<string> { "uno", "dos" };
var conO = lista.Where(s => s.Contains('o'));
var copia = conO.ToList();
lista.Add("cero");
Console.WriteLine($"{conO.Count()} {copia.Count}");
```

<details>
<summary>Solución</summary>

`3 2`. `conO` es diferida: al contar, vuelve a filtrar la lista, que ahora tiene "cero" (con 'o'). `copia` se materializó antes de agregar "cero", así que mantiene 2 elementos.

</details>

-----

## Siguiente lección

[Filtrar, ordenar y paginar](02-Filtrar%20ordenar%20y%20paginar.md)
