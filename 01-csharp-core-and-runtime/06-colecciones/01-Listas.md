# Listas

## En una frase

`List<T>` es una colección ordenada y **de tamaño dinámico**: agrega (`Add`), inserta (`Insert`), busca (`Contains`, `Find`, `IndexOf`) y elimina (`Remove`, `RemoveAt`) elementos por índice, sin que tengas que manejar la capacidad como en un array.

-----

## Antes de empezar

Conviene que ya sepas:

* Usar arrays, índices y bucles, de [Arrays](../02-control-de-flujo/03-Arrays.md) y [Bucles](../02-control-de-flujo/04-Bucles.md).
* Qué significa `T` en `List<T>`, de [Genéricos](../05-tipos-avanzados/04-Genericos.md).
* Lambdas y `Predicate<T>`, de [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`List<T>`:** colección genérica, ordenada y redimensionable.
* **`Count`:** cantidad de elementos que tiene la lista.
* **`Capacity`:** espacio reservado internamente; crece solo cuando hace falta.
* **Rango:** una porción contigua de elementos (desde un índice, cierta cantidad).
* **Inicializador de colección:** `new List<int> { 1, 2, 3 }`.
* **Expresión de colección:** `[1, 2, 3]` (C# 12).

-----

## El problema

Un array tiene un **tamaño fijo**. Si estás registrando los pedidos del día y no sabes cuántos van a llegar, tienes que:

```csharp
string[] pedidos = new string[10];
int cantidad = 0;

pedidos[cantidad++] = "Pedido A";
// ...y cuando llegue el pedido 11:
Array.Resize(ref pedidos, 20);        // copiar todo a un array más grande
```

Llevar a mano el contador, redimensionar, desplazar elementos al borrar uno del medio... es trabajo repetitivo y fácil de equivocar. `List<T>` hace todo eso por ti.

-----

## Cómo funciona

### Crear una lista

```csharp
List<string> ciudades = new List<string>();      // vacía
var numeros = new List<int>();                   // con var
List<int> otra = new();                          // new con tipo de destino (C# 9)

var colores = new List<string> { "Rojo", "Verde", "Azul" };   // inicializador de colección
List<string> frutas = ["Manzana", "Pera"];                     // expresión de colección (C# 12)

var copia = new List<string>(colores);           // a partir de otra colección
var grande = new List<int>(capacity: 1000);      // reserva espacio si sabes que serán muchos
```

`List<T>` vive en `System.Collections.Generic`, que ya está importado por defecto (`ImplicitUsings`).

### Agregar e insertar

```csharp
var ciudades = new List<string>();

ciudades.Add("Lima");                                  // al final
ciudades.Add("Cusco");
ciudades.Insert(0, "Arequipa");                        // en una posición: [Arequipa, Lima, Cusco]
ciudades.AddRange(new[] { "Piura", "Tacna" });         // varios al final
ciudades.InsertRange(1, new[] { "Ica" });              // varios en una posición
```

### Acceder y modificar

```csharp
string primera = ciudades[0];        // por índice, igual que un array
string ultima = ciudades[^1];        // desde el final
ciudades[1] = "Ica (Perú)";          // reemplazar

int cantidad = ciudades.Count;       // Count, no Length
```

Acceder a un índice que no existe lanza `ArgumentOutOfRangeException` (no `IndexOutOfRangeException` como en los arrays):

```csharp
var vacia = new List<int>();
int x = vacia[0];    // ArgumentOutOfRangeException: Index was out of range
```

### Buscar

```csharp
var numeros = new List<int> { 4, 8, 15, 16, 23, 42 };

bool tiene = numeros.Contains(15);              // true
int pos = numeros.IndexOf(16);                  // 3 (-1 si no está)
int primeroPar = numeros.Find(n => n % 2 == 0); // 4 (o default si no hay)
int idx = numeros.FindIndex(n => n > 20);       // 4
List<int> grandes = numeros.FindAll(n => n > 10);   // [15, 16, 23, 42]
bool hayNegativo = numeros.Exists(n => n < 0);  // false
bool todosPositivos = numeros.TrueForAll(n => n > 0);   // true
```

Igual que `Array.Find`, `List.Find` devuelve `default(T)` si nada coincide: con números, no distingues "encontré un 0" de "no encontré nada". Usa `FindIndex` o `Exists` cuando importe.

### Eliminar

```csharp
var ciudades = new List<string> { "Lima", "Cusco", "Piura", "Cusco" };

bool ok = ciudades.Remove("Cusco");      // quita la PRIMERA aparición; true si la encontró
bool no = ciudades.Remove("Quito");      // false: no existe (no lanza excepción)
ciudades.RemoveAt(0);                    // por índice
int quitados = ciudades.RemoveAll(c => c.StartsWith("P"));   // por condición; devuelve cuántos quitó
ciudades.Clear();                        // vacía la lista
```

Al quitar un elemento del medio, los siguientes se **desplazan** una posición hacia la izquierda: sus índices cambian.

### Rangos

```csharp
var lugares = new List<string> { "primero", "segundo", "quinto", "sexto" };

lugares.InsertRange(2, new[] { "tercero", "cuarto" });
// [primero, segundo, tercero, cuarto, quinto, sexto]

lugares.RemoveRange(4, 2);             // desde el índice 4, quita 2
// [primero, segundo, tercero, cuarto]

List<string> primeros = lugares.GetRange(0, 3);   // NUEVA lista: [primero, segundo, tercero]
List<string> slice = lugares[1..3];               // con rango (C# 12+/.NET 8): [segundo, tercero]
```

### Ordenar e invertir

```csharp
var nombres = new List<string> { "Luis", "Ana", "Eva" };

nombres.Sort();                                         // [Ana, Eva, Luis] (modifica la lista)
nombres.Sort((a, b) => b.Length.CompareTo(a.Length));   // con un comparador propio
nombres.Reverse();                                      // invierte en el lugar
```

`Sort` y `Reverse` **modifican** la lista. Si quieres conservar la original, ordena una copia o usa LINQ (`OrderBy`), que devuelve una secuencia nueva (ver [LINQ](../07-linq/README.md)).

### Recorrer

```csharp
foreach (string c in ciudades)                 // la opción por defecto
    Console.WriteLine(c);

for (int i = 0; i < ciudades.Count; i++)       // cuando necesitas el índice
    Console.WriteLine($"{i + 1}. {ciudades[i]}");

ciudades.ForEach(c => Console.WriteLine(c));   // método de List<T>
```

No agregues ni quites elementos dentro de un `foreach`: lanza `InvalidOperationException` (ver [Bucles](../02-control-de-flujo/04-Bucles.md)). Para quitar varios, usa `RemoveAll`.

### Cómo crece una lista por dentro

`List<T>` guarda los elementos en un **array interno**. Cuando se llena, crea uno del **doble** de tamaño y copia todo:

```csharp
var lista = new List<int>();
Console.WriteLine(lista.Capacity);   // 0
lista.Add(1);
Console.WriteLine(lista.Capacity);   // 4
for (int i = 0; i < 4; i++) lista.Add(i);
Console.WriteLine(lista.Capacity);   // 8
```

Por eso agregar al final es, en promedio, muy rápido, e insertar o quitar del **principio** es más lento: hay que desplazar todos los elementos. Si sabes de antemano cuántos elementos tendrás, pasa la capacidad en el constructor para evitar copias.

-----

## Ejemplo completo

Una lista de tareas en consola:

```csharp
var tareas = new List<string>();
bool salir = false;

while (!salir)
{
    Console.WriteLine();
    Console.WriteLine("1. Agregar  2. Listar  3. Completar  4. Buscar  5. Limpiar completadas  0. Salir");
    Console.Write("Opción: ");

    switch (Console.ReadLine())
    {
        case "1":
            Console.Write("Tarea: ");
            string? nueva = Console.ReadLine()?.Trim();
            if (string.IsNullOrEmpty(nueva)) break;
            if (tareas.Contains(nueva, StringComparer.OrdinalIgnoreCase))
                Console.WriteLine("Esa tarea ya existe.");
            else
                tareas.Add(nueva);
            break;

        case "2":
            if (tareas.Count == 0) { Console.WriteLine("No hay tareas."); break; }
            for (int i = 0; i < tareas.Count; i++)
                Console.WriteLine($"{i + 1}. {tareas[i]}");
            break;

        case "3":
            Console.Write("Número de tarea: ");
            if (int.TryParse(Console.ReadLine(), out int n) && n >= 1 && n <= tareas.Count)
                tareas[n - 1] = "[x] " + tareas[n - 1];
            else
                Console.WriteLine("Número inválido.");
            break;

        case "4":
            Console.Write("Buscar texto: ");
            string texto = Console.ReadLine() ?? "";
            var encontradas = tareas.FindAll(t => t.Contains(texto, StringComparison.OrdinalIgnoreCase));
            Console.WriteLine(encontradas.Count == 0 ? "Sin resultados." : string.Join(" | ", encontradas));
            break;

        case "5":
            int quitadas = tareas.RemoveAll(t => t.StartsWith("[x]"));
            Console.WriteLine($"Se quitaron {quitadas} tareas completadas.");
            break;

        case "0":
            salir = true;
            break;
    }
}
```

Ejecución de ejemplo:

```text
Opción: 1
Tarea: Estudiar listas
Opción: 1
Tarea: Hacer la práctica
Opción: 3
Número de tarea: 1
Opción: 2
1. [x] Estudiar listas
2. Hacer la práctica
Opción: 5
Se quitaron 1 tareas completadas.
```

`Contains` con `StringComparer.OrdinalIgnoreCase` es un método de LINQ que acepta un comparador; `List<T>.Contains` por sí solo distingue mayúsculas.

-----

## Errores comunes

**1. Índice fuera de rango.**
Qué pasa: `System.ArgumentOutOfRangeException: Index was out of range. Must be non-negative and less than the size of the collection.`
Por qué: accediste a un índice mayor o igual que `Count` (o la lista está vacía).
Arreglo: comprueba `if (i < lista.Count)` antes de acceder.

**2. Usar `Length` en una lista.**
Qué pasa: `error CS1061: 'List<int>' does not contain a definition for 'Length'`.
Por qué: los arrays usan `Length`; las colecciones, `Count`.
Arreglo: `lista.Count`.

**3. Modificar la lista dentro de un `foreach`.**
Qué pasa: `System.InvalidOperationException: Collection was modified; enumeration operation may not execute.`
Por qué: la lista detecta que cambió mientras se recorría.
Arreglo: `RemoveAll(condición)`, un `for` hacia atrás, o recorrer una copia (`lista.ToList()`).

**4. Esperar que `Remove` lance una excepción si no encuentra el elemento.**
Qué pasa: no pasa nada y el código sigue como si se hubiera borrado.
Por qué: `Remove` devuelve `false` en lugar de lanzar.
Arreglo: comprueba el `bool` que devuelve.

**5. Comparar listas con `==`.**
Qué pasa: dos listas con los mismos elementos dan `False`.
Por qué: `List<T>` es una clase: `==` compara referencias.
Arreglo: `lista1.SequenceEqual(lista2)` (LINQ).

**6. Quitar por índice dentro de un `for` hacia adelante.**
Qué pasa: se saltan elementos.
Por qué: al quitar el elemento `i`, el siguiente pasa a ser el `i` y el bucle avanza a `i + 1`.
Arreglo: recorre hacia atrás (`for (int i = lista.Count - 1; i >= 0; i--)`) o usa `RemoveAll`.

-----

## Según la versión de C#

* **C# 2 / .NET 2.0:** `List<T>`, que reemplazó a `ArrayList`.
* **C# 3:** inicializadores de colección (`new List<int> { 1, 2 }`).
* **C# 9:** `new()` con tipo de destino.
* **C# 12:** expresiones de colección: `List<int> l = [1, 2, 3];`, `[]` para una lista vacía y propagación `[.. a, .. b]`.
* **.NET 8:** `List<T>` admite rangos (`lista[1..3]`) y `CollectionsMarshal` para acceso de alto rendimiento.

-----

## Cuándo sí y cuándo no

**Usa `List<T>` cuando:**

* Necesitas una colección ordenada que crece y se reduce, con acceso por índice. Es la colección por defecto de .NET.

**Usa otra colección cuando:**

* Buscas por una clave (id, código): `Dictionary<TKey, TValue>`, que no recorre toda la colección.
* No quieres duplicados o haces operaciones de conjuntos: `HashSet<T>`.
* Siempre sacas del principio (cola) o del final (pila): `Queue<T>` o `Stack<T>`.
* El tamaño es fijo y conocido: un array.

Se comparan en [Elegir la colección adecuada](04-Elegir%20la%20coleccion%20adecuada.md).

-----

## Resumen en 5 líneas

1. `List<T>` es ordenada, accesible por índice y crece sola; `Count` da la cantidad.
2. `Add`, `Insert`, `AddRange`, `InsertRange` agregan; `Remove`, `RemoveAt`, `RemoveAll`, `Clear` quitan.
3. `Contains`, `IndexOf`, `Find`, `FindAll`, `Exists` buscan (`Find` devuelve `default` si no hay nada).
4. `Sort` y `Reverse` modifican la lista; `GetRange` y los rangos devuelven una lista nueva.
5. No la modifiques dentro de un `foreach`; por dentro es un array que duplica su capacidad al llenarse.

-----

## Para profundizar

<details>
<summary>Exponer listas sin perder el control</summary>

Si una clase expone su `List<T>` interna, cualquiera puede modificarla. Expón una vista de solo lectura:

```csharp
class Pedido
{
    private readonly List<string> _items = new();

    public IReadOnlyList<string> Items => _items;   // se puede leer y recorrer, no modificar
    public void Agregar(string item) => _items.Add(item);
}
```

Ojo: `IReadOnlyList<T>` impide modificar a través de esa referencia, pero quien la recibe podría hacer un cast a `List<T>`. Si necesitas una garantía total, devuelve `_items.AsReadOnly()` o una copia.

</details>

<details>
<summary>Complejidad de las operaciones</summary>

| Operación | Costo |
| --- | --- |
| Leer o escribir por índice | O(1) |
| `Add` al final | O(1) amortizado (O(n) cuando hay que crecer) |
| `Insert` / `RemoveAt` al principio o en el medio | O(n): desplaza elementos |
| `Contains`, `IndexOf`, `Remove(item)` | O(n): recorre la lista |
| `Sort` | O(n log n) |

"O(n)" significa que el tiempo crece en proporción a la cantidad de elementos. Si haces muchos `Contains` sobre una lista grande, un `HashSet<T>` (O(1)) es mucho más rápido.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`List<T>` es una colección genérica que, a diferencia de un array, puede crecer y reducirse. Tiene métodos como `Add`, `Remove`, `Insert`, `Contains`, `Find` y `Sort`, y se accede por índice. Su tamaño se consulta con `Count`.

### Respuesta ampliada (semi-senior)

`List<T>` se implementa sobre un array que duplica su capacidad al llenarse, así que `Add` es O(1) amortizado y el acceso por índice es O(1), pero `Insert`/`RemoveAt` en el medio y las búsquedas lineales (`Contains`, `IndexOf`) son O(n). Modificarla durante un `foreach` invalida el enumerador porque lleva un contador de versión. Conviene preasignar la capacidad cuando se conoce el tamaño, usar `RemoveAll` para borrados por condición y exponerla como `IReadOnlyList<T>` o `IEnumerable<T>` para no filtrar el estado interno. Para búsquedas por clave o pertenencia se prefiere `Dictionary` o `HashSet`.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre array y `List<T>`?**
El array tiene tamaño fijo; la lista crece y trae métodos para insertar, buscar y eliminar. Por dentro, la lista usa un array.

**2. ¿Qué pasa si llamas a `Remove` con un elemento que no existe?**
Devuelve `false`; no lanza excepción.

**3. ¿Cómo quitas elementos mientras recorres una lista?**
Con `RemoveAll(condición)`, con un `for` desde el final hacia el principio o recorriendo una copia.

-----

## Práctica

**Ejercicio 1.** Dada `var notas = new List<int> { 12, 18, 7, 15, 9, 20 };`, usa métodos de `List<T>` para: agregar un 14, quitar todas las notas menores a 10, insertar un 11 al principio, ordenar de mayor a menor y mostrar el resultado.

<details>
<summary>Solución</summary>

```csharp
var notas = new List<int> { 12, 18, 7, 15, 9, 20 };

notas.Add(14);
notas.RemoveAll(n => n < 10);
notas.Insert(0, 11);
notas.Sort((a, b) => b.CompareTo(a));

Console.WriteLine(string.Join(", ", notas));   // 20, 18, 15, 14, 12, 11
```

</details>

**Ejercicio 2.** Este código intenta quitar los números pares y no funciona bien. ¿Qué imprime y cómo lo arreglas?

```csharp
var numeros = new List<int> { 2, 4, 5, 6, 7 };
for (int i = 0; i < numeros.Count; i++)
{
    if (numeros[i] % 2 == 0) numeros.RemoveAt(i);
}
Console.WriteLine(string.Join(", ", numeros));
```

<details>
<summary>Solución</summary>

Imprime `4, 5, 7`. Al quitar el `2` (índice 0), el `4` pasa al índice 0, pero el bucle ya avanza al índice 1 y se lo salta. Arreglos:

```csharp
numeros.RemoveAll(n => n % 2 == 0);                 // la forma más clara

for (int i = numeros.Count - 1; i >= 0; i--)        // o recorrer hacia atrás
{
    if (numeros[i] % 2 == 0) numeros.RemoveAt(i);
}
```

</details>

-----

## Siguiente lección

[Diccionarios y conjuntos](02-Diccionarios%20y%20conjuntos.md)
