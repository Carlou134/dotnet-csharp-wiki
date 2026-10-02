# Elegir la colección adecuada

## En una frase

Cada colección está optimizada para un patrón de uso (acceso por índice, búsqueda por clave, unicidad, orden de llegada), las **interfaces** (`IEnumerable<T>`, `IReadOnlyList<T>`, `IList<T>`...) definen qué puede hacer quien recibe una colección, y las **colecciones heredadas** sin genéricos (`ArrayList`, `Hashtable`) no deberían usarse en código nuevo.

-----

## Antes de empezar

Conviene que ya sepas:

* Usar `List<T>`, `Dictionary`, `HashSet`, `Queue` y `Stack`, de las [lecciones anteriores](README.md).
* Qué es una interfaz y por qué conviene depender de ella, de [Interfaces](../04-poo/08-Interfaces.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Complejidad (notación O):** cómo crece el tiempo de una operación según la cantidad de elementos (n).
* **O(1):** tiempo constante: no depende de cuántos elementos haya.
* **O(log n):** crece muy lento (árboles, búsqueda binaria).
* **O(n):** crece en proporción a la cantidad de elementos (recorrer todo).
* **Colección heredada (*legacy*):** colección anterior a los genéricos, que guarda `object`.
* **Colección inmutable:** colección que no cambia; cada "modificación" devuelve una nueva.
* **Colección concurrente:** colección segura para usar desde varios hilos a la vez.

-----

## El problema

Usar siempre `List<T>` funciona... hasta que no. Un código real que llegó a producción:

```csharp
var procesados = new List<string>();           // 200.000 ids

foreach (var pedido in pedidosDelDia)          // 200.000 pedidos
{
    if (!procesados.Contains(pedido.Id))       // recorre la lista ENTERA cada vez
    {
        Procesar(pedido);
        procesados.Add(pedido.Id);
    }
}
```

Cada `Contains` recorre hasta 200.000 elementos, y se llama 200.000 veces: unas 20.000 millones de comparaciones. El proceso tardaba horas. Cambiar `List<string>` por `HashSet<string>` (búsqueda O(1)) lo bajó a menos de un segundo. **Una línea**.

Elegir la colección no es un detalle: define el rendimiento y lo que tu API permite (o no) hacer.

-----

## Cómo funciona

### Guía de decisión rápida

```text
¿Buscas por una clave (id, código)?            → Dictionary<TKey, TValue>
   ...y necesitas recorrer ordenado por clave? → SortedDictionary<TKey, TValue>
¿Necesitas valores únicos o saber "si está"?   → HashSet<T>   (ordenado: SortedSet<T>)
¿Procesas en orden de llegada?                  → Queue<T>
¿Lo último primero (deshacer, atrás)?           → Stack<T>
¿Lo más urgente primero?                        → PriorityQueue<T, P>
¿Tamaño fijo y conocido?                        → T[] (array)
¿Lista ordenada que crece, con índice?          → List<T>   ← la opción por defecto
```

### Complejidad de las operaciones habituales

| Colección | Acceso por índice | Buscar un elemento | Agregar | Quitar |
| --- | --- | --- | --- | --- |
| `T[]` | O(1) | O(n) | — (tamaño fijo) | — |
| `List<T>` | O(1) | O(n) | O(1)* al final | O(n) |
| `LinkedList<T>` | — | O(n) | O(1) en extremos o junto a un nodo | O(1) con el nodo |
| `Dictionary<K, V>` | — | **O(1)** por clave | O(1)* | O(1) |
| `HashSet<T>` | — | **O(1)** | O(1)* | O(1) |
| `SortedDictionary<K, V>` / `SortedSet<T>` | — | O(log n) | O(log n) | O(log n) |
| `Queue<T>` / `Stack<T>` | — | O(n) | O(1)* | O(1) del frente/tope |
| `PriorityQueue<T, P>` | — | — | O(log n) | O(log n) |

\* Amortizado: casi siempre O(1); ocasionalmente O(n) cuando el array interno tiene que crecer.

Para colecciones de pocas decenas de elementos, la diferencia no se nota. Importa cuando hay miles o millones, o cuando la operación está dentro de un bucle.

### Las interfaces de colecciones

```text
IEnumerable<T>                  "se puede recorrer" (foreach, LINQ)
   └── IReadOnlyCollection<T>   + Count
   │      └── IReadOnlyList<T>  + indexador de solo lectura
   └── ICollection<T>           + Count, Add, Remove, Contains, Clear
          ├── IList<T>          + indexador, Insert, RemoveAt
          ├── ISet<T>           + operaciones de conjuntos
          └── IDictionary<K, V> + acceso por clave
```

`List<T>` implementa casi todas; `HashSet<T>` implementa `ISet<T>`; un array implementa `IList<T>` e `IReadOnlyList<T>`.

### Qué tipo aceptar y qué tipo devolver

La regla: **acepta lo más general que necesites y devuelve lo más específico que tenga sentido, pero sin exponer más poder del necesario.**

```csharp
// ❌ Exige una List<T> aunque solo la recorre: no acepta arrays, HashSet, resultados de LINQ...
decimal Total(List<Producto> productos) { /* foreach */ }

// ✅ Acepta cualquier cosa recorrible
decimal Total(IEnumerable<Producto> productos) { /* foreach */ }

// ✅ Si necesitas Count o índice, pide justo eso
string Resumen(IReadOnlyList<Producto> productos) => $"{productos.Count} productos, el primero es {productos[0].Nombre}";
```

Para devolver datos internos de una clase, `IReadOnlyList<T>` o `IReadOnlyCollection<T>` comunican "puedes leer, no modificar":

```csharp
class Carrito
{
    private readonly List<Producto> _items = new();
    public IReadOnlyList<Producto> Items => _items;   // quien lo usa no puede hacer Add ni Clear
}
```

Cuidado con devolver `IEnumerable<T>` desde una consulta LINQ: puede volver a ejecutarse cada vez que se recorre (se ve en [LINQ](../07-linq/README.md)). Si el resultado ya está calculado, devuelve una colección concreta.

### Colecciones heredadas (no genéricas)

Antes de .NET 2.0 no existían los genéricos. Las colecciones de `System.Collections` guardan **`object`**:

| Heredada | Reemplazo moderno |
| --- | --- |
| `ArrayList` | `List<T>` |
| `Hashtable` | `Dictionary<TKey, TValue>` |
| `Queue`, `Stack` | `Queue<T>`, `Stack<T>` |
| `SortedList` | `SortedList<TKey, TValue>` / `SortedDictionary` |
| `ListDictionary`, `HybridDictionary` (`System.Collections.Specialized`) | `Dictionary<TKey, TValue>` |

Sus problemas:

```csharp
using System.Collections;

var lista = new ArrayList();
lista.Add(42);              // boxing: el int se convierte en object
lista.Add("hola");          // nada impide mezclar tipos

string s = (string)lista[0]!;   // compila, pero InvalidCastException al ejecutar: es un int

var tabla = new Hashtable();
tabla.Add("clave", 42);
string v = tabla["clave"];      // error CS0266: no se puede convertir 'object' a 'string' (hace falta un cast)
int n = (int)tabla["clave"]!;   // ✅ con cast... y unboxing
```

* **No hay seguridad de tipos:** los errores aparecen al ejecutar.
* **Hay que hacer casts** en cada lectura.
* **Boxing** de cada tipo de valor.

Solo las vas a encontrar en código antiguo o en APIs viejas. En código nuevo, usa siempre las genéricas.

### Colecciones especializadas

| Necesidad | Colección |
| --- | --- |
| Varios hilos leen y escriben a la vez | `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag` (`System.Collections.Concurrent`) |
| Datos que nunca cambian y se comparten | `ImmutableList`, `ImmutableDictionary` (`System.Collections.Immutable`) |
| Tabla de búsqueda creada al arrancar y solo leída | `FrozenDictionary`, `FrozenSet` (.NET 8) |
| Interfaz gráfica que se actualiza cuando la colección cambia | `ObservableCollection<T>` |
| Productor/consumidor asíncrono | `System.Threading.Channels.Channel<T>` |

### Expresiones de colección (C# 12)

La misma sintaxis sirve para casi todas las colecciones, y el compilador elige la implementación óptima:

```csharp
int[] array = [1, 2, 3];
List<string> lista = ["a", "b"];
HashSet<int> conjunto = [1, 2, 2, 3];        // { 1, 2, 3 }
IReadOnlyList<int> soloLectura = [4, 5];
Span<int> span = [1, 2, 3];

int[] unidos = [.. array, .. conjunto, 99];  // propagación
List<int> vacia = [];
```

-----

## Ejemplo completo

El mismo problema, "detectar pedidos duplicados", resuelto con `List<T>` y con `HashSet<T>`, midiendo el tiempo:

```csharp
using System.Diagnostics;

const int Cantidad = 30_000;
string[] ids = Enumerable.Range(0, Cantidad).Select(i => $"PED-{i % (Cantidad / 2)}").ToArray();
// cada id aparece dos veces → la mitad son duplicados

var reloj = Stopwatch.StartNew();
int duplicadosLista = ContarDuplicados(ids, new ListaComoRegistro());
Console.WriteLine($"List<T>:    {duplicadosLista} duplicados en {reloj.ElapsedMilliseconds} ms");

reloj.Restart();
int duplicadosSet = ContarDuplicados(ids, new ConjuntoComoRegistro());
Console.WriteLine($"HashSet<T>: {duplicadosSet} duplicados en {reloj.ElapsedMilliseconds} ms");

static int ContarDuplicados(IEnumerable<string> ids, IRegistro registro)
{
    int duplicados = 0;
    foreach (string id in ids)
    {
        if (!registro.Agregar(id)) duplicados++;
    }
    return duplicados;
}

interface IRegistro
{
    bool Agregar(string id);   // false si ya existía
}

class ListaComoRegistro : IRegistro
{
    private readonly List<string> _vistos = new();
    public bool Agregar(string id)
    {
        if (_vistos.Contains(id)) return false;   // O(n)
        _vistos.Add(id);
        return true;
    }
}

class ConjuntoComoRegistro : IRegistro
{
    private readonly HashSet<string> _vistos = new();
    public bool Agregar(string id) => _vistos.Add(id);   // O(1): Add ya devuelve false si existía
}
```

Salida aproximada (los tiempos dependen de la máquina):

```text
List<T>:    15000 duplicados en 1840 ms
HashSet<T>: 15000 duplicados en 2 ms
```

Mismo resultado, tres órdenes de magnitud de diferencia. Y gracias a la interfaz `IRegistro`, cambiar la implementación no tocó el método `ContarDuplicados`.

-----

## Errores comunes

**1. `Contains` sobre una lista grande dentro de un bucle.**
Qué pasa: el programa se vuelve muy lento con muchos datos, sin errores.
Por qué: O(n) dentro de O(n) es O(n²).
Arreglo: `HashSet<T>` para pertenencia o `Dictionary` para búsqueda por clave.

**2. Exigir `List<T>` en un parámetro que solo se recorre.**
Qué pasa: `error CS1503: Argument 1: cannot convert from 'string[]' to 'System.Collections.Generic.List<string>'` cuando alguien pasa un array.
Por qué: el parámetro pide más de lo que usa.
Arreglo: `IEnumerable<T>` o `IReadOnlyList<T>`.

**3. Exponer una `List<T>` interna.**
Qué pasa: código externo hace `Clear()` y rompe el estado del objeto.
Por qué: devolviste la referencia mutable.
Arreglo: expón `IReadOnlyList<T>` o una copia.

**4. Usar colecciones heredadas sin cast.**
Qué pasa: `error CS0266: Cannot implicitly convert type 'object' to 'string'. An explicit conversion exists (are you missing a cast?)`.
Por qué: `Hashtable` y `ArrayList` devuelven `object`.
Arreglo: migra a `Dictionary`/`List<T>`; si no puedes, haz el cast con `is`/`as` para evitar `InvalidCastException`.

**5. Usar `Dictionary` o `List` desde varios hilos.**
Qué pasa: datos corruptos o excepciones aleatorias (por ejemplo, `InvalidOperationException` o índices inconsistentes).
Por qué: las colecciones comunes no son seguras para escritura concurrente.
Arreglo: `ConcurrentDictionary`, `ConcurrentQueue` o un `lock`.

-----

## Según la versión de C#

* **.NET 1.x:** solo colecciones no genéricas (`ArrayList`, `Hashtable`).
* **.NET 2.0:** colecciones genéricas.
* **.NET 4.0:** colecciones concurrentes.
* **.NET 4.5:** interfaces `IReadOnlyList<T>` e `IReadOnlyCollection<T>`.
* **.NET 6:** `PriorityQueue<T, P>`.
* **.NET 8:** `FrozenDictionary` y `FrozenSet`.
* **C# 12:** expresiones de colección `[...]` y propagación `..`.

-----

## Cuándo sí y cuándo no

**Empieza por `List<T>`** y cambia cuando el patrón de uso lo pida: búsquedas por clave, unicidad, orden de llegada o prioridad.

**En tus firmas públicas:**

* Parámetros: `IEnumerable<T>` si solo recorres; `IReadOnlyList<T>` o `IReadOnlyCollection<T>` si necesitas `Count` o índice.
* Retornos: una colección concreta o una interfaz de solo lectura; evita devolver `null` (devuelve una colección vacía).

**No optimices sin medir:** con 20 elementos, cualquier colección sirve. Elige primero por claridad y cambia cuando haya un problema real o una cantidad de datos que lo justifique.

-----

## Resumen en 5 líneas

1. `List<T>` por defecto; `Dictionary` para buscar por clave; `HashSet` para unicidad; `Queue`/`Stack`/`PriorityQueue` para el orden de salida.
2. Buscar en una lista es O(n); en un `Dictionary`/`HashSet`, O(1): importa dentro de bucles con muchos datos.
3. Acepta `IEnumerable<T>` o `IReadOnlyList<T>` en tus parámetros; expón colecciones internas como solo lectura.
4. `ArrayList` y `Hashtable` son heredadas: sin seguridad de tipos, con casts y boxing. No las uses en código nuevo.
5. Para concurrencia, inmutabilidad o datos congelados existen colecciones especializadas.

-----

## Para profundizar

<details>
<summary>Por qué O(n²) explota</summary>

Si una operación O(n) está dentro de un bucle de n vueltas, el total es n × n = n². Con 1.000 elementos son 1.000.000 de pasos (rápido); con 100.000 son 10.000.000.000 (minutos u horas). El crecimiento es cuadrático: duplicar los datos cuadruplica el tiempo. Aprender a reconocer este patrón (un `Contains`, `IndexOf`, `Find` o `Where` dentro de un `foreach`) es una de las habilidades más rentables para escribir código eficiente.

</details>

<details>
<summary>Medir en serio: BenchmarkDotNet</summary>

`Stopwatch` sirve para una idea aproximada, pero las mediciones de rendimiento serias tienen en cuenta el calentamiento del JIT, la recolección de basura y la variación entre ejecuciones. La herramienta estándar en .NET es **BenchmarkDotNet** (`dotnet add package BenchmarkDotNet`), que ejecuta cada caso muchas veces y entrega estadísticas y la memoria asignada.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Depende del uso: `List<T>` para una lista general con índice, `Dictionary` para buscar por clave, `HashSet` para elementos únicos, `Queue` para orden de llegada y `Stack` para el último primero. Las colecciones como `ArrayList` y `Hashtable` son antiguas, no tienen seguridad de tipos y se reemplazaron por las genéricas.

### Respuesta ampliada (semi-senior)

La elección se basa en el patrón de acceso y en la complejidad: búsqueda lineal O(n) en `List<T>` frente a O(1) promedio en las tablas hash y O(log n) en las colecciones ordenadas (árboles). En las APIs se acepta la abstracción mínima necesaria (`IEnumerable<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>`) y se exponen vistas de solo lectura para proteger el estado. Las colecciones no genéricas implican boxing y casts y solo se toleran por compatibilidad. Para multihilo existen `System.Collections.Concurrent` y `Channels`; para inmutabilidad, `System.Collections.Immutable`; y para lectura intensiva de datos estáticos, `FrozenDictionary`. Cualquier optimización se valida midiendo con BenchmarkDotNet.

### Preguntas frecuentes de seguimiento

**1. ¿Qué colección usarías para comprobar si un elemento ya fue procesado?**
`HashSet<T>`: `Add` devuelve `false` si ya estaba, en O(1).

**2. ¿Qué tipo usarías como parámetro de un método que solo recorre elementos?**
`IEnumerable<T>`, para aceptar cualquier colección.

**3. ¿Por qué no usar `ArrayList`?**
Porque guarda `object`: no tiene seguridad de tipos, requiere casts y hace boxing de los tipos de valor.

-----

## Práctica

**Ejercicio 1.** Elige la colección más adecuada para cada caso y justifica:

1. Los 12 meses del año.
2. Un historial de páginas visitadas con botón "atrás".
3. Un catálogo de productos que se consulta por código de barras.
4. Las etiquetas de un artículo de blog (sin repetir).
5. Los pedidos que esperan ser despachados en orden de llegada.

<details>
<summary>Solución</summary>

1. **Array** (`string[]`): tamaño fijo y conocido.
2. **`Stack<string>`**: "atrás" devuelve la última página visitada.
3. **`Dictionary<string, Producto>`**: búsqueda por clave en O(1).
4. **`HashSet<string>`**: evita duplicados automáticamente.
5. **`Queue<Pedido>`**: FIFO.

</details>

**Ejercicio 2.** Mejora la firma de este método para que acepte arrays, listas y conjuntos, y explica por qué es mejor:

```csharp
static int ContarMayores(List<int> numeros, int limite)
{
    int c = 0;
    foreach (var n in numeros) if (n > limite) c++;
    return c;
}
```

<details>
<summary>Solución</summary>

```csharp
static int ContarMayores(IEnumerable<int> numeros, int limite)
{
    int c = 0;
    foreach (var n in numeros) if (n > limite) c++;
    return c;
}

Console.WriteLine(ContarMayores(new[] { 1, 5, 9 }, 4));            // 2 (array)
Console.WriteLine(ContarMayores(new List<int> { 10, 2 }, 4));      // 1 (lista)
Console.WriteLine(ContarMayores(new HashSet<int> { 7, 8 }, 4));    // 2 (conjunto)
```

El método solo recorre los números, así que `IEnumerable<int>` es todo lo que necesita. Pedir `List<int>` obligaba a quien llama a convertir sus datos a una lista sin ningún motivo.

</details>

-----

## Siguiente lección

[Iteradores y yield](05-Iteradores%20y%20yield.md)
