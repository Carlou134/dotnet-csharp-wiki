# Diccionarios y conjuntos

## En una frase

`Dictionary<TKey, TValue>` asocia **claves únicas con valores** y encuentra un valor por su clave casi al instante; `HashSet<T>` guarda **elementos únicos** y hace operaciones de conjuntos (unión, intersección, diferencia); sus versiones `SortedDictionary` y `SortedSet` además los mantienen **ordenados**.

-----

## Antes de empezar

Conviene que ya sepas:

* Usar `List<T>`, de [Listas](01-Listas.md).
* Qué son `Equals` y `GetHashCode` y por qué van juntos, de [La clase Object](../04-poo/10-La%20clase%20Object.md).
* El patrón `TryX(..., out valor)`, de [Valores de retorno y parámetros out](../03-metodos/03-Valores%20de%20retorno%20y%20parametros%20out.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Diccionario:** colección de pares clave-valor donde cada clave es única.
* **Clave (*key*):** el dato por el que se busca (un id, un código, un nombre).
* **Valor (*value*):** el dato asociado a la clave.
* **`KeyValuePair<TKey, TValue>`:** un par clave-valor al recorrer un diccionario.
* **Tabla hash:** estructura que usa `GetHashCode` para ubicar un elemento sin recorrer todos.
* **Conjunto (*set*):** colección sin duplicados.
* **Unión / intersección / diferencia:** todos los elementos / los comunes / los que están en uno y no en el otro.

-----

## El problema

Tienes una red de 10.000 sensores, cada uno con un identificador como `"sensor_living"`, y quieres la lectura de uno en particular. Con una lista:

```csharp
var lecturas = new List<(string Id, int Valor)> { /* 10.000 elementos */ };
var lectura = lecturas.Find(l => l.Id == "sensor_bedroom");   // recorre hasta encontrarlo
```

En el peor caso, revisas los 10.000 elementos, y lo haces **cada vez** que buscas. Además, nada impide tener dos sensores con el mismo id.

Y otro problema: recibes tres listas de correos de distintas campañas y quieres saber cuáles son **únicos**, cuáles aparecen en **todas** y cuáles **solo en una**. Con listas, son bucles anidados y comparaciones a mano.

-----

## Cómo funciona

### `Dictionary<TKey, TValue>`

```csharp
var lecturas = new Dictionary<string, int>();

lecturas["sensor_living"] = 10;        // agrega (o reemplaza si ya existe)
lecturas["sensor_bano"] = 20;
lecturas.Add("sensor_dormitorio", 30); // agrega; lanza excepción si la clave ya existe

int valor = lecturas["sensor_living"]; // 10: búsqueda directa por clave
lecturas["sensor_living"] = 12;        // actualiza

Console.WriteLine(lecturas.Count);     // 3
```

Con inicializador:

```csharp
var extensiones = new Dictionary<string, string>
{
    [".doc"] = "Word",
    [".xls"] = "Excel",
    [".pdf"] = "Acrobat"
};
```

### Buscar sin excepciones: `TryGetValue`

Leer una clave que no existe con `[]` **lanza una excepción**:

```csharp
int x = lecturas["sensor_cocina"];   // KeyNotFoundException
```

La forma segura:

```csharp
if (lecturas.TryGetValue("sensor_cocina", out int lectura))
{
    Console.WriteLine($"Cocina: {lectura}");
}
else
{
    Console.WriteLine("Sensor no encontrado");
}

int conDefecto = lecturas.GetValueOrDefault("sensor_cocina", -1);   // -1 si no existe
```

`TryGetValue` hace **una sola** búsqueda. Hacer `if (ContainsKey(k)) { var v = dic[k]; }` hace dos.

### Agregar sin pisar: `TryAdd`

```csharp
bool agregado = extensiones.TryAdd(".doc", "Otro Word");   // false: la clave ya existe, no cambia nada
extensiones.Add(".doc", "Otro Word");                       // ArgumentException: la clave ya existe
extensiones[".doc"] = "Otro Word";                          // reemplaza sin preguntar
```

| Forma | Si la clave **no** existe | Si la clave **ya** existe |
| --- | --- | --- |
| `dic[k] = v` | Agrega | Reemplaza |
| `dic.Add(k, v)` | Agrega | Lanza `ArgumentException` |
| `dic.TryAdd(k, v)` | Agrega y devuelve `true` | Devuelve `false` |

### Otras operaciones

```csharp
bool existe = extensiones.ContainsKey(".pdf");          // rápido (por clave)
bool hayExcel = extensiones.ContainsValue("Excel");     // LENTO: recorre todos los valores
bool quitado = extensiones.Remove(".xls");              // true si existía

foreach (KeyValuePair<string, string> par in extensiones)
{
    Console.WriteLine($"{par.Key} → {par.Value}");
}

foreach (var (ext, programa) in extensiones)            // deconstrucción del par
{
    Console.WriteLine($"{ext} → {programa}");
}

foreach (string clave in extensiones.Keys) { }          // solo claves
foreach (string programa in extensiones.Values) { }     // solo valores
```

**El orden de un `Dictionary` no está garantizado.** En la práctica suele coincidir con el orden de inserción si no borraste nada, pero no debes depender de eso. Si necesitas orden, usa `SortedDictionary` o ordena al recorrer.

### Comparadores de claves

Por defecto, las claves `string` distinguen mayúsculas. Puedes pasar un comparador al crear el diccionario:

```csharp
var usuarios = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase)
{
    ["Ana"] = 1
};

Console.WriteLine(usuarios.ContainsKey("ANA"));   // True
```

### Qué hace buena a una clave

El diccionario usa `GetHashCode()` para encontrar el "casillero" y `Equals()` para confirmar:

```text
dic["Ana"]
   │
   ▼
"Ana".GetHashCode() → 1834   →   1834 % 8 casilleros = casillero 2
                                                        │
 casillero 0: (vacío)                                   │
 casillero 1: "Luis" → 7                                │
 casillero 2: "Eva" → 3, "Ana" → 5   ◄──────────────────┘  compara con Equals: "Eva"? no. "Ana"? sí → 5
 casillero 3: (vacío)
 ...
```

No recorre todo el diccionario: va directo a un casillero y compara solo las pocas claves que hay ahí. Por eso:

* Las claves deben implementar bien `Equals` y `GetHashCode` (los tipos primitivos, `string`, los records y las tuplas ya lo hacen).
* Las claves **no deben cambiar** después de agregarlas: si cambia su hash, el diccionario no la vuelve a encontrar.
* Una clave no puede ser `null`.

```csharp
var porUbicacion = new Dictionary<(string Pasillo, int Estante), int>
{
    [("A", 1)] = 30,
    [("B", 4)] = 12
};
Console.WriteLine(porUbicacion[("A", 1)]);   // 30: las tuplas sirven como clave compuesta
```

### `HashSet<T>`: elementos únicos

```csharp
var correos = new HashSet<string>();

bool a = correos.Add("ana@mail.com");    // true
bool b = correos.Add("ana@mail.com");    // false: ya estaba, no se duplica
correos.Add("luis@mail.com");

Console.WriteLine(correos.Count);                    // 2
Console.WriteLine(correos.Contains("luis@mail.com")); // True: rápido, sin recorrer
correos.Remove("ana@mail.com");
```

Quitar duplicados de una lista en una línea:

```csharp
var conRepetidos = new List<int> { 3, 1, 3, 2, 1 };
var unicos = new HashSet<int>(conRepetidos);   // { 3, 1, 2 }
```

### Operaciones de conjuntos

```text
      A = {1,2,3,4,5}        B = {3,4,5,6,7}

     ┌───────────┬───────┬───────────┐
     │   1   2   │ 3 4 5 │   6   7   │
     └───────────┴───────┴───────────┘
       solo en A   ambos   solo en B

UnionWith            → 1 2 3 4 5 6 7     (todo)
IntersectWith        → 3 4 5             (el centro)
ExceptWith           → 1 2               (solo en A)
SymmetricExceptWith  → 1 2 6 7           (los extremos)
```

Cada operación **modifica** el conjunto sobre el que se llama:

```csharp
var campañaA = new HashSet<int> { 1, 2, 3, 4, 5 };
var campañaB = new HashSet<int> { 3, 4, 5, 6, 7 };

var union = new HashSet<int>(campañaA);
union.UnionWith(campañaB);                    // { 1, 2, 3, 4, 5, 6, 7 }

var comunes = new HashSet<int>(campañaA);
comunes.IntersectWith(campañaB);              // { 3, 4, 5 }

var soloA = new HashSet<int>(campañaA);
soloA.ExceptWith(campañaB);                   // { 1, 2 }

var enUnoSolo = new HashSet<int>(campañaA);
enUnoSolo.SymmetricExceptWith(campañaB);      // { 1, 2, 6, 7 }

bool esSub = new HashSet<int> { 3, 4 }.IsSubsetOf(campañaA);   // True
bool seSolapan = campañaA.Overlaps(campañaB);                   // True
```

Por eso se crea una copia antes de cada operación: si encadenaras `UnionWith` y después `IntersectWith` sobre `campañaA`, la intersección se calcularía sobre el resultado de la unión.

### Colecciones ordenadas

```csharp
var ranking = new SortedDictionary<string, int>
{
    ["Luis"] = 85,
    ["Ana"] = 95,
    ["Eva"] = 90
};

foreach (var (nombre, puntaje) in ranking)
    Console.WriteLine($"{nombre}: {puntaje}");   // Ana, Eva, Luis: ordenado por CLAVE

var tickets = new SortedSet<int> { 30, 5, 12, 5 };
Console.WriteLine(string.Join(", ", tickets));   // 5, 12, 30 (únicos y ordenados)
Console.WriteLine(tickets.Min);                  // 5
Console.WriteLine(tickets.Max);                  // 30
```

| Colección | Orden | Búsqueda | Úsala cuando... |
| --- | --- | --- | --- |
| `Dictionary<K, V>` | No garantizado | Muy rápida (hash) | Buscas por clave y el orden no importa |
| `SortedDictionary<K, V>` | Por clave | Rápida (árbol) | Necesitas recorrer en orden de clave |
| `HashSet<T>` | No garantizado | Muy rápida | Quieres unicidad o pertenencia |
| `SortedSet<T>` | Ordenado | Rápida | Únicos, ordenados, con `Min`/`Max` |

-----

## Ejemplo completo

Contar palabras de un texto y analizar vocabulario:

```csharp
string texto1 = "el perro come y el gato duerme y el perro ladra";
string texto2 = "el gato come y el pájaro canta";

// 1. Frecuencia de palabras con Dictionary
var frecuencias = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
foreach (string palabra in texto1.Split(' ', StringSplitOptions.RemoveEmptyEntries))
{
    frecuencias[palabra] = frecuencias.GetValueOrDefault(palabra) + 1;
}

Console.WriteLine("Frecuencias (texto 1):");
foreach (var (palabra, veces) in new SortedDictionary<string, int>(frecuencias))
{
    Console.WriteLine($"  {palabra,-8} {new string('*', veces)} ({veces})");
}

// 2. Vocabulario con HashSet
var vocab1 = new HashSet<string>(texto1.Split(' '));
var vocab2 = new HashSet<string>(texto2.Split(' '));

var comunes = new HashSet<string>(vocab1);
comunes.IntersectWith(vocab2);

var soloEn1 = new HashSet<string>(vocab1);
soloEn1.ExceptWith(vocab2);

Console.WriteLine($"Palabras únicas en total: {vocab1.Union(vocab2).Count()}");
Console.WriteLine($"Comunes: {string.Join(", ", new SortedSet<string>(comunes))}");
Console.WriteLine($"Solo en el texto 1: {string.Join(", ", new SortedSet<string>(soloEn1))}");

// 3. Búsqueda segura
string buscada = "pájaro";
Console.WriteLine(frecuencias.TryGetValue(buscada, out int n)
    ? $"'{buscada}' aparece {n} veces"
    : $"'{buscada}' no aparece en el texto 1");
```

Salida:

```text
Frecuencias (texto 1):
  come     * (1)
  duerme   * (1)
  el       *** (3)
  gato     * (1)
  ladra    * (1)
  perro    ** (2)
  y        ** (2)
Palabras únicas en total: 9
Comunes: come, el, gato, y
Solo en el texto 1: duerme, ladra, perro
'pájaro' no aparece en el texto 1
```

`frecuencias[palabra] = frecuencias.GetValueOrDefault(palabra) + 1` es el patrón clásico de conteo: si la palabra no existe, parte de 0. (`vocab1.Union(vocab2)` es la versión de LINQ: devuelve una secuencia nueva sin modificar los conjuntos).

-----

## Errores comunes

**1. Leer una clave que no existe.**
Qué pasa: `System.Collections.Generic.KeyNotFoundException: The given key 'sensor_cocina' was not present in the dictionary.`
Por qué: el indexador `[]` exige que la clave exista.
Arreglo: `TryGetValue` o `GetValueOrDefault`.

**2. `Add` con una clave repetida.**
Qué pasa: `System.ArgumentException: An item with the same key has already been added. Key: .doc`.
Por qué: las claves son únicas.
Arreglo: `TryAdd` si no quieres pisar, o `dic[k] = v` si quieres reemplazar.

**3. Modificar el diccionario mientras se recorre.**
Qué pasa: `InvalidOperationException: Collection was modified`.
Por qué: igual que con las listas.
Arreglo: junta las claves a borrar en una lista y bórralas después del `foreach`.

**4. Depender del orden de un `Dictionary` o un `HashSet`.**
Qué pasa: el orden "parece" estable en tus pruebas y cambia en producción tras algunos borrados.
Por qué: no está garantizado.
Arreglo: `SortedDictionary`/`SortedSet`, o ordena al mostrar.

**5. Usar como clave un objeto que cambia.**
Qué pasa: `ContainsKey` devuelve `false` para una clave que "está" en el diccionario.
Por qué: su `GetHashCode` cambió después de agregarla.
Arreglo: usa claves inmutables (strings, números, records con propiedades `init`).

**6. Usar `ContainsValue` en diccionarios grandes.**
Qué pasa: rendimiento pobre, sin errores.
Por qué: recorre todos los valores (O(n)).
Arreglo: si buscas por ese dato con frecuencia, mantén un segundo diccionario indexado por él.

-----

## Según la versión de C#

* **.NET 2.0:** `Dictionary<TKey, TValue>` y, desde .NET 3.5, `HashSet<T>`; reemplazaron a `Hashtable`.
* **C# 6:** inicializadores de índice (`["clave"] = valor`).
* **.NET Core 2.0:** `TryAdd`, `GetValueOrDefault` y la deconstrucción de `KeyValuePair` (`var (k, v)`).
* **C# 12:** expresiones de colección para `HashSet<T>` (`HashSet<int> s = [1, 2, 3];`).
* **.NET 8:** `FrozenDictionary` y `FrozenSet`, optimizados para datos que se crean una vez y solo se leen.

-----

## Cuándo sí y cuándo no

**Usa `Dictionary` cuando:**

* Buscas por una clave: productos por código, usuarios por id, configuración por nombre, conteos.

**Usa `HashSet` cuando:**

* Necesitas saber rápido si algo "ya está" (visitados, procesados) o eliminar duplicados.
* Haces operaciones de conjuntos.

**Usa las versiones `Sorted...` cuando:**

* Además necesitas recorrer en orden o consultar mínimos y máximos con frecuencia.

**No uses un diccionario cuando:**

* Siempre recorres todos los elementos y nunca buscas por clave: una lista es más simple.

-----

## Resumen en 5 líneas

1. `Dictionary<K, V>` busca por clave sin recorrer la colección; las claves son únicas y no `null`.
2. `dic[k]` lanza `KeyNotFoundException` si no existe: usa `TryGetValue` o `GetValueOrDefault`.
3. `Add` lanza si la clave existe; `TryAdd` devuelve `false`; `dic[k] = v` reemplaza.
4. `HashSet<T>` guarda únicos; `UnionWith`, `IntersectWith`, `ExceptWith` modifican el conjunto.
5. `Dictionary` y `HashSet` no garantizan orden; `SortedDictionary` y `SortedSet` sí.

-----

## Para profundizar

<details>
<summary>Cómo funciona una tabla hash</summary>

Al agregar una clave, el diccionario calcula `clave.GetHashCode()` y, con ese número, elige un "casillero" (*bucket*). Para buscar, recalcula el hash, va directo a ese casillero y compara con `Equals` las pocas claves que haya ahí. Por eso la búsqueda es O(1) en promedio, sin importar cuántos elementos haya. Si muchas claves distintas tienen el mismo hash (colisiones), un casillero se llena y la búsqueda se degrada. Un buen `GetHashCode` reparte bien los valores; `HashCode.Combine` lo hace por ti.

</details>

<details>
<summary>Colecciones especializadas</summary>

* `SortedList<K, V>`: como `SortedDictionary`, pero sobre arrays; usa menos memoria y es más rápida para datos que casi no cambian.
* `ConcurrentDictionary<K, V>`: segura para varios hilos a la vez (se ve con la asincronía).
* `ImmutableDictionary<K, V>`: cada modificación devuelve un diccionario nuevo.
* `FrozenDictionary<K, V>` (.NET 8): para tablas de búsqueda que se crean al arrancar y nunca cambian; las lecturas son aún más rápidas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un `Dictionary<TKey, TValue>` guarda pares clave-valor con claves únicas y permite buscar un valor por su clave muy rápido. Para leer de forma segura se usa `TryGetValue`. Un `HashSet<T>` guarda elementos sin duplicados y permite operaciones de conjuntos como unión e intersección. Ninguno de los dos garantiza el orden; para eso están `SortedDictionary` y `SortedSet`.

### Respuesta ampliada (semi-senior)

`Dictionary` y `HashSet` son tablas hash: inserción, búsqueda y borrado O(1) promedio, dependientes de un `GetHashCode` bien distribuido y coherente con `Equals`; las claves deben ser inmutables mientras estén en la colección. Se puede pasar un `IEqualityComparer<T>` (por ejemplo, `StringComparer.OrdinalIgnoreCase`). `TryGetValue` evita la doble búsqueda de `ContainsKey` + indexador. `SortedDictionary`/`SortedSet` son árboles rojo-negro con O(log n) y orden garantizado. Para concurrencia existe `ConcurrentDictionary`, y para datos de solo lectura en caliente, `FrozenDictionary` (.NET 8).

### Preguntas frecuentes de seguimiento

**1. ¿Qué pasa si agregas una clave duplicada con `Add`?**
Lanza `ArgumentException`. Con el indexador, en cambio, reemplaza el valor.

**2. ¿Por qué un `Dictionary` es más rápido que una lista para buscar?**
Porque usa el hash de la clave para ir directo a su posición (O(1)), en lugar de recorrer todos los elementos (O(n)).

**3. ¿Qué requisitos tiene un tipo para ser clave?**
Implementar `Equals` y `GetHashCode` de forma coherente, no ser `null` y no cambiar mientras está en el diccionario.

-----

## Práctica

**Ejercicio 1.** Dada una lista de ventas `(string Producto, int Cantidad)`, usa un `Dictionary<string, int>` para calcular el total vendido por producto y muestra el resultado ordenado por nombre.

```csharp
var ventas = new List<(string Producto, int Cantidad)>
{
    ("Mouse", 2), ("Teclado", 1), ("Mouse", 3), ("Monitor", 1), ("Teclado", 4)
};
```

<details>
<summary>Solución</summary>

```csharp
var totales = new Dictionary<string, int>();
foreach (var (producto, cantidad) in ventas)
{
    totales[producto] = totales.GetValueOrDefault(producto) + cantidad;
}

foreach (var (producto, total) in new SortedDictionary<string, int>(totales))
{
    Console.WriteLine($"{producto}: {total}");
}
// Monitor: 1
// Mouse: 5
// Teclado: 5
```

</details>

**Ejercicio 2.** Tienes los asistentes de dos días de un evento. Con `HashSet<string>`, muestra: quiénes fueron los dos días, quiénes fueron solo el primero y cuántas personas distintas asistieron en total.

```csharp
string[] dia1 = { "Ana", "Luis", "Eva", "Juan" };
string[] dia2 = { "Eva", "Juan", "Rosa" };
```

<details>
<summary>Solución</summary>

```csharp
var ambos = new HashSet<string>(dia1);
ambos.IntersectWith(dia2);

var soloDia1 = new HashSet<string>(dia1);
soloDia1.ExceptWith(dia2);

var todos = new HashSet<string>(dia1);
todos.UnionWith(dia2);

Console.WriteLine($"Ambos días: {string.Join(", ", ambos)}");          // Eva, Juan
Console.WriteLine($"Solo el día 1: {string.Join(", ", soloDia1)}");     // Ana, Luis
Console.WriteLine($"Personas distintas: {todos.Count}");               // 5
```

</details>

-----

## Siguiente lección

[Colas, pilas y listas enlazadas](03-Colas%20pilas%20y%20listas%20enlazadas.md)
