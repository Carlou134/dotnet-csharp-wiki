# Arrays

## En una frase

Un array es una colección de **tamaño fijo** de elementos del **mismo tipo**, guardados en orden y accesibles por un **índice** que empieza en 0.

-----

## Antes de empezar

Conviene que ya sepas:

* Declarar variables y que los tipos de referencia se copian por referencia, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).
* Indexar un string con `texto[0]`, de [Texto: char y string](../01-tipos-y-variables/05-Texto%20char%20y%20string.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Array (arreglo):** estructura que guarda varios valores del mismo tipo en posiciones consecutivas.
* **Elemento:** cada valor dentro del array.
* **Índice:** la posición de un elemento. El primero es el 0 y el último es `Length - 1`.
* **Estructura de datos:** forma organizada de guardar y acceder a datos.
* **Predicado:** función que recibe un elemento y devuelve `bool` (por ejemplo, `x => x > 5`).
* **Array multidimensional:** array con varias dimensiones, como una tabla (`int[,]`).
* **Array escalonado (*jagged*):** array de arrays, donde cada fila puede tener distinto largo (`int[][]`).

-----

## El problema

Quieres registrar la altura de las plantas de tu jardín:

```csharp
int plantaUno = 4;
int plantaDos = 3;
int plantaTres = 6;
```

¿Y si tienes 50 plantas? ¿Cómo calculas el promedio sin escribir 50 sumas? ¿Cómo encuentras la más alta? Con variables sueltas es imposible de mantener. Necesitas **un solo nombre** que agrupe todos los valores y te permita recorrerlos por posición.

-----

## Cómo funciona

### Declarar y crear

El tipo de un array es el tipo de sus elementos seguido de `[]`:

```csharp
int[] alturas;          // declara una variable de tipo "array de int" (todavía sin array)
string[] nombres;
double[] precios;
```

Formas de crear e inicializar:

```csharp
int[] a = { 3, 4, 6 };                   // inicializador: el tamaño lo dan los elementos
int[] b = new int[] { 3, 4, 6 };         // con new explícito
int[] c = new int[3];                    // tamaño 3, todos con el valor por defecto: [0, 0, 0]
int[] d = [3, 4, 6];                     // expresión de colección (C# 12)
var e = new[] { 3, 4, 6 };               // tipo inferido: int[]
```

Si declaras primero y asignas después, `{ ... }` solo no sirve:

```csharp
int[] alturas;
alturas = { 3, 4, 6 };            // error CS0623: los inicializadores de array solo se pueden usar en una declaración
alturas = new int[] { 3, 4, 6 };  // ✅
alturas = [3, 4, 6];              // ✅ desde C# 12
```

### Valores por defecto

Un array creado con un tamaño se llena con el valor por defecto del tipo:

| Tipo de elemento | Valor inicial |
| --- | --- |
| `int`, `double`, `decimal` | `0` |
| `bool` | `false` |
| `char` | `'\0'` |
| `string` y cualquier clase | `null` |

### Longitud y acceso por índice

```csharp
int[] alturas = { 3, 4, 6 };

Console.WriteLine(alturas.Length);   // 3
Console.WriteLine(alturas[0]);       // 3  primer elemento
Console.WriteLine(alturas[1]);       // 4
Console.WriteLine(alturas[2]);       // 6  último: Length - 1
Console.WriteLine(alturas[^1]);      // 6  último, contando desde el final
```

```text
índice:    0    1    2
         ┌────┬────┬────┐
alturas  │ 3  │ 4  │ 6  │     Length = 3
         └────┴────┴────┘
desde el final:  ^3   ^2   ^1
```

Acceder a una posición inexistente falla **al ejecutar**:

```csharp
Console.WriteLine(alturas[3]);   // System.IndexOutOfRangeException
```

### Modificar elementos

El **tamaño es fijo**, pero los valores se pueden cambiar:

```csharp
int[] alturas = new int[3];   // [0, 0, 0]
alturas[2] = 8;               // [0, 0, 8]
alturas[0] = 5;               // [5, 0, 8]
alturas[1]++;                 // [5, 1, 8]
```

No existe `alturas.Add(10)`: un array no crece. Si necesitas agregar y quitar elementos, usa `List<T>`, que se ve en [Listas](../06-colecciones/01-Listas.md).

### Rangos: obtener una porción

```csharp
int[] numeros = { 10, 20, 30, 40, 50 };

int[] medio = numeros[1..4];    // [20, 30, 40]  (el 4 no se incluye)
int[] primeros = numeros[..2];  // [10, 20]
int[] ultimos = numeros[^2..];  // [40, 50]
```

En arrays, un rango crea un **array nuevo** (una copia de esa porción).

### Los arrays son tipos de referencia

```csharp
int[] original = { 1, 2, 3 };
int[] otro = original;     // misma referencia, mismo array
otro[0] = 99;

Console.WriteLine(original[0]);   // 99

int[] copia = (int[])original.Clone();   // copia real (superficial)
copia[0] = 1;
Console.WriteLine(original[0]);   // 99, no cambió
```

### Métodos de la clase `Array`

```csharp
int[] alturas = { 3, 6, 4, 1, 6, 8 };

Array.Sort(alturas);                               // ordena EN el mismo array: [1, 3, 4, 6, 6, 8]
Array.Reverse(alturas);                            // invierte: [8, 6, 6, 4, 3, 1]

int pos = Array.IndexOf(alturas, 6);               // 1  primera aparición (o -1 si no está)
int primeraAlta = Array.Find(alturas, h => h > 5); // 8  primer elemento que cumple
int posAlta = Array.FindIndex(alturas, h => h > 5);// 0
int[] altas = Array.FindAll(alturas, h => h > 5);  // [8, 6, 6]
bool hayPar = Array.Exists(alturas, h => h % 2 == 0); // true

Array.Clear(alturas, 1, 2);                        // pone en 0 dos elementos desde el índice 1
Array.Resize(ref alturas, 10);                     // crea un array NUEVO de 10 y copia los valores
```

* `Sort`, `Reverse` y `Clear` **modifican** el array.
* `Find` recibe un **predicado**, normalmente una lambda (`h => h > 5`). Las lambdas se estudian en [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).
* `Find` devuelve el **valor por defecto** (`0` para `int`) si nada coincide. Si 0 puede ser un valor válido, usa `FindIndex` y comprueba `-1`.
* `Resize` no "estira" el array: crea uno nuevo y lo asigna a la variable. Por eso necesita `ref`.

La lista completa está en la [documentación de `System.Array`](https://learn.microsoft.com/dotnet/api/system.array).

### Strings y arrays

```csharp
char[] letras = "Hola".ToCharArray();            // ['H', 'o', 'l', 'a']
string[] partes = "a,b,c".Split(',');            // ["a", "b", "c"]
string unido = string.Join("-", partes);         // "a-b-c"
string desdeChars = new string(letras);          // "Hola"
```

### Arrays de varias dimensiones

**Multidimensional (rectangular):** una tabla donde todas las filas tienen el mismo largo.

```csharp
int[,] tablero = new int[3, 3];       // 3 filas, 3 columnas
tablero[0, 2] = 1;

int[,] matriz =
{
    { 1, 2, 3 },
    { 4, 5, 6 }
};
Console.WriteLine(matriz[1, 2]);           // 6
Console.WriteLine(matriz.GetLength(0));    // 2 filas
Console.WriteLine(matriz.GetLength(1));    // 3 columnas
```

**Escalonado (*jagged*):** un array de arrays, donde cada fila puede tener un largo distinto.

```csharp
int[][] semanas = new int[3][];
semanas[0] = new int[] { 1, 2 };
semanas[1] = new int[] { 3, 4, 5, 6 };
semanas[2] = new int[] { 7 };

Console.WriteLine(semanas[1][3]);   // 6
```

-----

## Ejemplo completo

```csharp
string[] plantas = { "Helecho", "Cactus", "Bambú", "Orquídea", "Menta" };
int[] alturas = { 30, 12, 150, 25, 18 };

// Recorrer con for para usar el índice en los dos arrays
// (los bucles se ven en detalle en la próxima lección)
int suma = 0;
int indiceMasAlta = 0;

for (int i = 0; i < alturas.Length; i++)
{
    suma += alturas[i];
    if (alturas[i] > alturas[indiceMasAlta])
    {
        indiceMasAlta = i;
    }
}

double promedio = (double)suma / alturas.Length;
int[] grandes = Array.FindAll(alturas, a => a > promedio);
int posCactus = Array.IndexOf(plantas, "Cactus");

int[] ordenadas = (int[])alturas.Clone();
Array.Sort(ordenadas);

Console.WriteLine($"Plantas registradas: {plantas.Length}");
Console.WriteLine($"Promedio: {promedio:F1} cm");
Console.WriteLine($"Más alta: {plantas[indiceMasAlta]} ({alturas[indiceMasAlta]} cm)");
Console.WriteLine($"Sobre el promedio: {string.Join(", ", grandes)}");
Console.WriteLine($"El cactus mide {alturas[posCactus]} cm");
Console.WriteLine($"Alturas ordenadas: {string.Join(", ", ordenadas)}");
Console.WriteLine($"Original intacto: {string.Join(", ", alturas)}");
```

Salida:

```text
Plantas registradas: 5
Promedio: 47.0 cm
Más alta: Bambú (150 cm)
Sobre el promedio: 150
El cactus mide 12 cm
Alturas ordenadas: 12, 18, 25, 30, 150
Original intacto: 30, 12, 150, 25, 18
```

Se ordenó una **copia** (`Clone`) porque `Array.Sort` modifica el array y se habría perdido la relación entre `plantas[i]` y `alturas[i]`.

-----

## Errores comunes

**1. Acceder al índice `Length`.**
Qué pasa: `System.IndexOutOfRangeException: Index was outside the bounds of the array.`
Por qué: los índices van de `0` a `Length - 1`.
Arreglo: en los bucles usa `i < arr.Length`, no `i <= arr.Length`.

**2. Usar `{ }` para asignar después de declarar.**
Qué pasa: `error CS0623: Array initializers can only be used in a variable or field initializer. Try using a new expression instead.`
Por qué: el inicializador corto solo funciona en la declaración.
Arreglo: `arr = new int[] { 1, 2 };` o `arr = [1, 2];` (C# 12).

**3. Mezclar tipos.**
Qué pasa: `int[] x = { 1, "dos" };` da `error CS0029: Cannot implicitly convert type 'string' to 'int'`.
Por qué: todos los elementos deben ser del tipo del array.
Arreglo: usa un solo tipo, o modela los datos con una clase.

**4. Creer que asignar copia el array.**
Qué pasa: modificas `copia` y cambia `original`.
Por qué: los arrays son tipos de referencia.
Arreglo: `Clone()`, `[..original]` o `original.ToArray()`.

**5. Confiar en el resultado de `Array.Find` sin comprobar.**
Qué pasa: `Find` devuelve `0` (o `null`) y no sabes si encontró un 0 o no encontró nada.
Por qué: devuelve `default(T)` cuando no hay coincidencias.
Arreglo: `FindIndex` y comprobar `-1`, o `Exists` antes.

**6. Usar un elemento `null` de un array de strings.**
Qué pasa: `NullReferenceException` en `nombres[2].Length`.
Por qué: `new string[5]` crea 5 posiciones con `null`, no con `""`.
Arreglo: inicializa los elementos antes de usarlos.

-----

## Según la versión de C#

* **C# 3:** `new[] { ... }` con el tipo inferido.
* **C# 8:** índices desde el final (`arr[^1]`) y rangos (`arr[1..3]`).
* **C# 12:** expresiones de colección: `int[] a = [1, 2, 3];`, `a = [];` y el operador de propagación `[.. a, .. b]` para unir arrays.

-----

## Cuándo sí y cuándo no

**Usa un array cuando:**

* La cantidad de elementos es fija y conocida (los 12 meses, las casillas de un tablero).
* Necesitas el máximo rendimiento en acceso por índice.
* Una API lo exige (`Main(string[] args)`, `Split`).

**Usa `List<T>` cuando:**

* Vas a agregar o quitar elementos. En la práctica, es la colección que más vas a usar.

**Usa una clase o un record cuando:**

* Tienes arrays "paralelos" (`nombres[i]` y `edades[i]`): agrupa los datos en un tipo `Persona` y usa un solo array o lista.

-----

## Resumen en 5 líneas

1. `int[] a = { 1, 2, 3 };` crea un array; `new int[n]` crea n elementos con el valor por defecto.
2. Los índices van de `0` a `Length - 1`; `a[^1]` es el último y `a[1..3]` es una porción.
3. El tamaño es fijo: puedes cambiar valores, no agregar elementos (para eso existe `List<T>`).
4. Los arrays son tipos de referencia: asignar comparte el mismo array; `Clone()` copia.
5. `Array.Sort`, `IndexOf`, `Find`, `FindAll` y `Exists` resuelven las operaciones comunes.

-----

## Para profundizar

<details>
<summary>Copia superficial frente a copia profunda</summary>

`Clone()` crea un array nuevo, pero si los elementos son objetos (tipos de referencia), copia las **referencias**, no los objetos:

```csharp
var personas = new[] { new Persona { Nombre = "Ana" } };
var copia = (Persona[])personas.Clone();
copia[0].Nombre = "Luis";
Console.WriteLine(personas[0].Nombre);   // Luis: el array es otro, pero el objeto es el mismo
```

Eso es una **copia superficial**. Para una copia profunda hay que crear un objeto nuevo por cada elemento.

</details>

<details>
<summary>Covarianza de arrays: una trampa histórica</summary>

C# permite asignar un `string[]` a una variable `object[]`. Compila, pero puede fallar al ejecutar:

```csharp
object[] objetos = new string[2];
objetos[0] = 42;   // ArrayTypeMismatchException
```

Es un diseño heredado de Java. Las colecciones genéricas modernas (`List<T>`) no tienen este problema.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un array guarda una cantidad fija de elementos del mismo tipo, a los que se accede por un índice que empieza en 0. Su tamaño no cambia después de crearlo. `Length` da la cantidad de elementos, y acceder fuera del rango lanza `IndexOutOfRangeException`. Para colecciones que crecen se usa `List<T>`.

### Respuesta ampliada (semi-senior)

Los arrays son tipos de referencia que derivan de `System.Array`, ocupan memoria contigua en el heap y ofrecen acceso O(1) por índice con verificación de límites. Al crearlos se inicializan con `default(T)`. Los arrays multidimensionales (`int[,]`) son un solo bloque rectangular; los escalonados (`int[][]`) son arrays de arrays, a menudo más rápidos de recorrer. `Clone()` hace una copia superficial. Desde C# 8 existen `Index` y `Range`, y desde C# 12, las expresiones de colección con propagación. `List<T>` usa internamente un array que se redimensiona al llenarse.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `int[,]` e `int[][]`?**
`int[,]` es rectangular: un bloque con filas del mismo largo. `int[][]` es un array de arrays: cada fila es independiente y puede tener otro largo.

**2. ¿Qué devuelve `Array.IndexOf` si no encuentra el valor?**
El límite inferior menos 1, que en la práctica es `-1`.

**3. ¿Por qué `Array.Resize` necesita `ref`?**
Porque crea un array nuevo y tiene que actualizar la variable del llamador para que apunte a él.

-----

## Práctica

**Ejercicio 1.** Sin ejecutar, ¿qué imprime?

```csharp
int[] a = new int[4];
a[1] = 5;
a[^1] = 9;
int[] b = a;
b[0] = 7;
Console.WriteLine(string.Join(",", a));
Console.WriteLine(a[1..3].Length);
```

<details>
<summary>Solución</summary>

```text
7,5,0,9
2
```

`b` y `a` son el mismo array, por eso `b[0] = 7` se ve en `a`. El rango `1..3` incluye los índices 1 y 2: dos elementos.

</details>

**Ejercicio 2.** Dado `int[] temperaturas = { 18, 25, 31, 12, 27, 33, 20 };`, usa métodos de `Array` para obtener: la primera temperatura mayor a 30, cuántas superan 24 y el array ordenado de menor a mayor **sin modificar el original**.

<details>
<summary>Solución</summary>

```csharp
int[] temperaturas = { 18, 25, 31, 12, 27, 33, 20 };

int primeraMayor30 = Array.Find(temperaturas, t => t > 30);          // 31
int cantidadMayor24 = Array.FindAll(temperaturas, t => t > 24).Length; // 4

int[] ordenadas = (int[])temperaturas.Clone();
Array.Sort(ordenadas);

Console.WriteLine(primeraMayor30);
Console.WriteLine(cantidadMayor24);
Console.WriteLine(string.Join(", ", ordenadas));   // 12, 18, 20, 25, 27, 31, 33
```

</details>

-----

## Siguiente lección

[Bucles](04-Bucles.md)
