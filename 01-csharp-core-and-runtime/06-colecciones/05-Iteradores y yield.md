# Iteradores y yield

## En una frase

Un **iterador** es un método que devuelve `IEnumerable<T>` y usa `yield return` para entregar los elementos **de a uno y solo cuando se piden**, de modo que puedes generar secuencias enormes o infinitas sin construirlas completas en memoria (evaluación perezosa).

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo funciona `foreach` y que, por dentro, usa `GetEnumerator`, `MoveNext` y `Current`, de [Bucles](../02-control-de-flujo/04-Bucles.md).
* Interfaces genéricas, de [Genéricos](../05-tipos-avanzados/04-Genericos.md), y la jerarquía de interfaces de colecciones, de [Elegir la colección adecuada](04-Elegir%20la%20coleccion%20adecuada.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`IEnumerable<T>`:** "algo que se puede recorrer". Tiene un único método: `GetEnumerator()`.
* **`IEnumerator<T>`:** el "cursor" que recorre: `MoveNext()`, `Current` y `Dispose()`.
* **Iterador:** método, propiedad o indexador que usa `yield return` o `yield break`.
* **`yield return`:** entrega un elemento y **pausa** el método hasta que se pida el siguiente.
* **`yield break`:** termina la secuencia.
* **Evaluación perezosa (*lazy*) / ejecución diferida:** el código no se ejecuta al llamar al método, sino al recorrer el resultado.
* **Máquina de estados:** la clase que genera el compilador para recordar en qué punto quedó el iterador.

-----

## El problema

Necesitas los números primos que hagan falta, pero no sabes cuántos. O procesar un archivo de registros de 20 GB línea por línea. Con lo que sabes hasta ahora:

```csharp
static List<int> PrimerosPrimos(int cantidad)
{
    var primos = new List<int>();
    for (int n = 2; primos.Count < cantidad; n++)
        if (EsPrimo(n)) primos.Add(n);
    return primos;   // toda la lista se calcula y se guarda en memoria ANTES de devolverla
}
```

Tienes que decidir de antemano cuántos calcular, y guardarlos todos en memoria aunque quien llama solo use el primero. Con el archivo de 20 GB, `File.ReadAllLines` directamente no entra en memoria.

Lo que quieres es una secuencia que **produzca cada elemento cuando se lo piden**, y que se detenga cuando quien la consume deja de pedir.

-----

## Cómo funciona

### `yield return`: entregar de a uno

```csharp
static IEnumerable<int> Numeros()
{
    Console.WriteLine("  [inicio]");
    yield return 1;
    Console.WriteLine("  [después del 1]");
    yield return 2;
    Console.WriteLine("  [después del 2]");
    yield return 3;
    Console.WriteLine("  [fin]");
}
```

```csharp
var secuencia = Numeros();          // ¡no imprime nada! El método todavía no empezó
Console.WriteLine("Antes del foreach");

foreach (int n in secuencia)
{
    Console.WriteLine($"Recibí {n}");
}
```

Salida:

```text
Antes del foreach
  [inicio]
Recibí 1
  [después del 1]
Recibí 2
  [después del 2]
Recibí 3
  [fin]
```

```text
   foreach (consumidor)                 Numeros() (iterador)
   ────────────────────                 ────────────────────
   MoveNext() ───────────────────────►  [inicio] ... yield return 1  ⏸ pausa
   Current = 1  ◄─────────────────────
   "Recibí 1"
   MoveNext() ───────────────────────►  reanuda ... yield return 2  ⏸ pausa
   Current = 2  ◄─────────────────────
   "Recibí 2"
   MoveNext() ───────────────────────►  reanuda ... yield return 3  ⏸ pausa
   ...
   MoveNext() ───────────────────────►  reanuda ... [fin]
   false        ◄─────────────────────  termina el foreach
```

Lo que pasa:

1. Llamar a `Numeros()` **no ejecuta** el cuerpo: solo crea el objeto que sabe recorrerlo.
2. Cada vuelta del `foreach` llama a `MoveNext()`, que **reanuda** el método desde donde quedó hasta el siguiente `yield return`.
3. `yield return` entrega el valor y **pausa** el método, guardando sus variables locales.
4. Cuando el método llega al final, `MoveNext()` devuelve `false` y el `foreach` termina.

### `yield break`: terminar antes

```csharp
static IEnumerable<int> HastaLimite(int limite)
{
    for (int i = 0; ; i++)
    {
        if (i >= limite) yield break;   // termina la secuencia
        yield return i;
    }
}

Console.WriteLine(string.Join(", ", HastaLimite(4)));   // 0, 1, 2, 3
```

En un iterador no se puede usar `return valor;`; para terminar se usa `yield break` (o llegar al final del método).

### Secuencias infinitas

Como los elementos se generan bajo demanda, una secuencia puede no tener fin. Quien la consume decide cuándo parar:

```csharp
static IEnumerable<int> Primos()
{
    for (int n = 2; ; n++)
    {
        if (EsPrimo(n)) yield return n;
    }
}

static bool EsPrimo(int n)
{
    if (n < 2) return false;
    for (int d = 2; d * d <= n; d++)
        if (n % d == 0) return false;
    return true;
}

foreach (int p in Primos())
{
    if (p > 30) break;            // el consumidor corta
    Console.Write($"{p} ");       // 2 3 5 7 11 13 17 19 23 29
}

var cinco = Primos().Take(5).ToList();          // con LINQ: [2, 3, 5, 7, 11]
var desde100 = Primos().SkipWhile(p => p < 100).First();   // 101
```

`Take(5)` pide solo cinco elementos; el iterador nunca calcula el sexto. Sin `Take` o `break`, un `foreach` sobre `Primos()` no terminaría nunca, y `Primos().ToList()` se quedaría sin memoria.

### Leer un archivo enorme línea por línea

```csharp
static IEnumerable<string> LeerLineas(string ruta)
{
    using var lector = new StreamReader(ruta);
    string? linea;
    while ((linea = lector.ReadLine()) is not null)
    {
        yield return linea;
    }
}   // el using cierra el archivo cuando termina el recorrido... o cuando se corta con break

foreach (string linea in LeerLineas("app.log"))
{
    if (linea.Contains("ERROR")) Console.WriteLine(linea);
}
```

En memoria hay **una línea a la vez**, sin importar el tamaño del archivo. (.NET ya trae este patrón: `File.ReadLines`, a diferencia de `File.ReadAllLines`, que carga todo; se ve en el módulo de archivos).

### Componer iteradores

Un iterador puede consumir otros iteradores y transformarlos, como una cadena de montaje:

```csharp
static IEnumerable<int> Pares(IEnumerable<int> origen)
{
    foreach (int n in origen)
        if (n % 2 == 0) yield return n;
}

static IEnumerable<int> AlCuadrado(IEnumerable<int> origen)
{
    foreach (int n in origen)
        yield return n * n;
}

var resultado = AlCuadrado(Pares(Enumerable.Range(1, 10)));
Console.WriteLine(string.Join(", ", resultado));   // 4, 16, 36, 64, 100
```

Cada número viaja por toda la cadena antes de que se procese el siguiente. **Así funciona LINQ por dentro**: `Where` y `Select` son iteradores como estos (ver [LINQ](../07-linq/README.md)).

También sirve para aplanar estructuras anidadas:

```csharp
static IEnumerable<int> Aplanar(IEnumerable<IEnumerable<int>> listas)
{
    foreach (var lista in listas)
        foreach (int item in lista)
            yield return item;
}
```

### Lo que genera el compilador

Un iterador se compila a una **clase oculta** que implementa `IEnumerable<T>` e `IEnumerator<T>`: guarda las variables locales como campos y un número de "estado" que indica en qué `yield` quedó. Escribirlo a mano se vería así:

```csharp
class Contador : IEnumerable<int>
{
    private readonly int _hasta;
    public Contador(int hasta) => _hasta = hasta;

    public IEnumerator<int> GetEnumerator() => new Enumerador(_hasta);
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();

    private class Enumerador : IEnumerator<int>
    {
        private readonly int _hasta;
        private int _actual = -1;

        public Enumerador(int hasta) => _hasta = hasta;

        public int Current => _actual;
        object System.Collections.IEnumerator.Current => Current;

        public bool MoveNext() => ++_actual < _hasta;
        public void Reset() => _actual = -1;
        public void Dispose() { }
    }
}

foreach (int n in new Contador(3)) Console.Write(n);   // 012
```

Con `yield`, lo mismo es:

```csharp
static IEnumerable<int> Contar(int hasta)
{
    for (int i = 0; i < hasta; i++) yield return i;
}
```

| Miembro de `IEnumerator<T>` | Qué hace |
| --- | --- |
| `MoveNext()` | Avanza al siguiente elemento; devuelve `false` si no hay más. |
| `Current` | El elemento actual. |
| `Reset()` | Vuelve al principio (los iteradores con `yield` no lo admiten: lanzan `NotSupportedException`). |
| `Dispose()` | Libera recursos; `foreach` lo llama siempre al terminar. |

Fíjate en que `foreach` recorre un **`IEnumerable<T>`** (algo que tiene `GetEnumerator()`), no un `IEnumerator<T>` directamente. Implementar solo el enumerador no alcanza para usar `foreach`.

-----

## Ejemplo completo

Un generador de datos de prueba y un procesador por lotes, todo perezoso:

```csharp
var ventas = GenerarVentas(semilla: 7);                     // secuencia infinita: todavía no se generó nada

int lote = 1;
foreach (var grupo in EnLotes(ventas.Take(10), tamano: 4))  // solo se generarán 10 ventas
{
    decimal total = grupo.Sum(v => v.Monto);
    Console.WriteLine($"Lote {lote++}: {grupo.Count} ventas, total {total:N2}");
}

var primeraGrande = GenerarVentas(semilla: 7).First(v => v.Monto > 900);
Console.WriteLine($"Primera venta mayor a 900: #{primeraGrande.Id} por {primeraGrande.Monto:N2}");

static IEnumerable<Venta> GenerarVentas(int semilla)
{
    var random = new Random(semilla);
    for (int id = 1; ; id++)
    {
        yield return new Venta(id, Math.Round((decimal)random.NextDouble() * 1000, 2));
    }
}

static IEnumerable<List<T>> EnLotes<T>(IEnumerable<T> origen, int tamano)
{
    var lote = new List<T>(tamano);
    foreach (var item in origen)
    {
        lote.Add(item);
        if (lote.Count == tamano)
        {
            yield return lote;
            lote = new List<T>(tamano);   // lista nueva: no reutilizar la ya entregada
        }
    }
    if (lote.Count > 0) yield return lote;   // el último lote incompleto
}

record Venta(int Id, decimal Monto);
```

Salida con esta forma (los montos y el número de venta concretos dependen de la implementación de `Random`; lo importante es la estructura):

```text
Lote 1: 4 ventas, total 2,184.35
Lote 2: 4 ventas, total 1,707.92
Lote 3: 2 ventas, total 1,129.10
Primera venta mayor a 900: #4 por 932.17
```

`GenerarVentas` es infinita, pero `Take(10)` y `First(...)` piden solo lo necesario. `EnLotes` es un iterador genérico reutilizable (.NET 6 ya trae uno: `Chunk`).

-----

## Errores comunes

**1. Usar `return valor;` en un iterador.**
Qué pasa: `error CS1622: Cannot return a value from an iterator. Use the yield return statement to return a value, or yield break to end the iteration.`
Por qué: en un método con `yield`, el compilador genera la secuencia; no puedes devolver otra cosa.
Arreglo: `yield return valor;` o `yield break;`.

**2. Esperar que el iterador se ejecute al llamarlo.**
Qué pasa: un iterador que valida sus argumentos no lanza la excepción al llamarlo, sino mucho después, al recorrerlo.
Por qué: ejecución diferida: nada corre hasta el primer `MoveNext()`.
Arreglo: valida en un método normal y delega en una función local iteradora (ver "Para profundizar").

**3. Recorrer varias veces una secuencia costosa o con efectos.**
Qué pasa: el trabajo se repite (o los números aleatorios cambian) en cada recorrido.
Por qué: cada `foreach`, `Count()` o `ToList()` vuelve a ejecutar el iterador desde el principio.
Arreglo: si vas a recorrerla más de una vez, materializa una vez con `.ToList()`.

**4. `ToList()` o `Count()` sobre una secuencia infinita.**
Qué pasa: el programa se cuelga y termina con `OutOfMemoryException`.
Por qué: esos métodos recorren hasta el final, que nunca llega.
Arreglo: limita antes con `Take`, `TakeWhile` o `First`.

**5. `yield return` dentro de un `try` con `catch`.**
Qué pasa: `error CS1626: Cannot yield a value in the body of a try block with a catch clause`.
Por qué: limitación de la máquina de estados (sí se permite `try` con `finally`).
Arreglo: maneja la excepción en un bloque separado y haz el `yield return` fuera del `try`.

**6. Hacer `foreach` sobre un `IEnumerator<T>`.**
Qué pasa: `error CS1579: foreach statement cannot operate on variables of type 'SaveFileReader' because 'SaveFileReader' does not contain a public instance or extension definition for 'GetEnumerator'`.
Por qué: `foreach` necesita algo recorrible (`GetEnumerator()`), no el cursor.
Arreglo: implementa `IEnumerable<T>` (o, mucho más simple, escribe un método con `yield`).

-----

## Según la versión de C#

* **C# 2:** iteradores con `yield return` y `yield break`.
* **C# 3:** LINQ, construido sobre iteradores y ejecución diferida.
* **C# 7:** funciones locales, el patrón recomendado para validar argumentos de un iterador.
* **C# 8:** iteradores asíncronos: `async IAsyncEnumerable<T>` con `await foreach` (se ven en el módulo de asincronía).
* **.NET 6:** `Chunk`, el equivalente incorporado del `EnLotes` del ejemplo.

-----

## Cuándo sí y cuándo no

**Usa un iterador cuando:**

* Generas datos bajo demanda, potencialmente infinitos o costosos de calcular.
* Procesas fuentes grandes (archivos, resultados paginados) sin cargarlas completas.
* Escribes transformaciones encadenables sobre secuencias.

**Materializa (`ToList`, `ToArray`) cuando:**

* Vas a recorrer el resultado más de una vez o necesitas `Count` o índices.
* La fuente puede cambiar o cerrarse (una conexión, un archivo) antes de que se recorra.

-----

## Resumen en 5 líneas

1. Un método con `yield return` devuelve `IEnumerable<T>` y entrega los elementos de a uno, pausándose entre cada uno.
2. Llamarlo no ejecuta nada: el código corre al recorrer (ejecución diferida); `yield break` termina.
3. Permite secuencias infinitas o enormes; el consumidor corta con `break`, `Take` o `First`.
4. Cada recorrido vuelve a ejecutar el iterador: materializa con `ToList()` si lo recorres varias veces.
5. El compilador genera una máquina de estados que implementa `IEnumerable<T>`/`IEnumerator<T>`; `foreach` necesita `GetEnumerator()`.

-----

## Para profundizar

<details>
<summary>Validar argumentos sin perder la ejecución diferida</summary>

```csharp
static IEnumerable<int> Rango(int desde, int cantidad)
{
    ArgumentOutOfRangeException.ThrowIfNegative(cantidad);   // se ejecuta AL LLAMAR
    return Iterar();

    IEnumerable<int> Iterar()                                // función local iteradora
    {
        for (int i = 0; i < cantidad; i++)
            yield return desde + i;
    }
}

var r = Rango(1, -5);   // lanza aquí, en la llamada, no cuando alguien la recorra
```

`Rango` no usa `yield`, así que se ejecuta inmediatamente; la parte perezosa vive en la función local. Así están implementados los operadores de LINQ.

</details>

<details>
<summary>try/finally y Dispose en iteradores</summary>

Si un iterador tiene `using` o `finally`, ese código se ejecuta cuando el recorrido termina **o cuando se corta** (con `break`, `First`, `Take`, una excepción), porque `foreach` llama a `Dispose()` sobre el enumerador, y el `Dispose` generado ejecuta los `finally` pendientes. Por eso `LeerLineas` cierra el archivo aunque el `foreach` salga antes. Si recorres a mano con `GetEnumerator()`/`MoveNext()`, envuelve el enumerador en un `using`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un iterador es un método que devuelve `IEnumerable<T>` y usa `yield return` para devolver los elementos de uno en uno. Se ejecuta de forma perezosa: el código corre recién cuando se recorre el resultado, por ejemplo con un `foreach`. Así se pueden generar secuencias muy grandes o infinitas sin cargarlas en memoria.

### Respuesta ampliada (semi-senior)

El compilador transforma un método con `yield` en una máquina de estados que implementa `IEnumerable<T>` e `IEnumerator<T>`, con las variables locales elevadas a campos. La ejecución es diferida: nada se ejecuta hasta el primer `MoveNext()`, por lo que la validación de argumentos se separa con una función local para que falle en la llamada. Cada enumeración vuelve a ejecutar el iterador, lo que puede repetir trabajo o efectos (el problema de la enumeración múltiple). Los bloques `finally` y `using` se ejecutan en `Dispose()`, que `foreach` garantiza incluso al cortar. LINQ to Objects está construido con iteradores encadenados; la versión asíncrona es `IAsyncEnumerable<T>` con `await foreach`.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `IEnumerable<T>` e `IEnumerator<T>`?**
`IEnumerable<T>` es algo recorrible: sabe crear un enumerador. `IEnumerator<T>` es el cursor que avanza (`MoveNext`) y expone el elemento actual (`Current`).

**2. ¿Qué es la ejecución diferida?**
Que el código de una secuencia no se ejecuta al definirla, sino al recorrerla, y se vuelve a ejecutar en cada recorrido.

**3. ¿Qué diferencia hay entre `yield return` y devolver una `List<T>`?**
La lista se calcula completa y se guarda en memoria antes de devolverse. Con `yield`, cada elemento se calcula cuando se pide, y se puede cortar antes.

-----

## Práctica

**Ejercicio 1.** Escribe un iterador `Fibonacci()` infinito y úsalo para mostrar los términos menores que 100.

<details>
<summary>Solución</summary>

```csharp
foreach (long f in Fibonacci())
{
    if (f >= 100) break;
    Console.Write($"{f} ");   // 0 1 1 2 3 5 8 13 21 34 55 89
}

static IEnumerable<long> Fibonacci()
{
    long a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}
```

Con LINQ: `Fibonacci().TakeWhile(f => f < 100)`.

</details>

**Ejercicio 2.** ¿Qué imprime este programa? Piensa en la ejecución diferida.

```csharp
var nums = Generar();
Console.WriteLine("A");
Console.WriteLine(nums.First());
Console.WriteLine(nums.First());

static IEnumerable<int> Generar()
{
    Console.WriteLine("generando");
    yield return 1;
    yield return 2;
}
```

<details>
<summary>Solución</summary>

```text
A
generando
1
generando
1
```

`Generar()` no imprime nada al llamarlo. Cada `First()` recorre la secuencia **desde el principio**, así que "generando" aparece dos veces.

</details>

-----

## Siguiente lección

Terminaste el módulo de colecciones. Continúa con [LINQ](../07-linq/README.md).
