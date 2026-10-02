# Bucles

## En una frase

Un bucle repite un bloque de código: `while` mientras se cumpla una condición, `do...while` igual pero al menos una vez, `for` una cantidad conocida de veces y `foreach` una vez por cada elemento de una colección; `break`, `continue` y `return` alteran ese recorrido.

-----

## Antes de empezar

Conviene que ya sepas:

* Escribir condiciones booleanas, de [Lógica booleana](01-Logica%20booleana.md).
* Crear arrays y acceder por índice, de [Arrays](03-Arrays.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Bucle (*loop*):** estructura que repite un bloque de código.
* **Iteración:** cada repetición del bucle.
* **Condición de parada:** la condición que, al volverse `false`, termina el bucle.
* **Variable de control (iterador):** la variable que cuenta las vueltas, como `i` en un `for`.
* **Bucle infinito:** un bucle cuya condición nunca se vuelve `false`.
* **Colección:** cualquier grupo de elementos que se puede recorrer (array, lista, string...).
* **Sentencia de salto:** `break`, `continue`, `return`, que cambian el flujo normal del bucle.

-----

## El problema

Estás programando un videojuego y quieres agregar 15 alienígenas a la pantalla:

```csharp
AgregarAlien();
AgregarAlien();
AgregarAlien();
// ... 12 veces más
```

Copiar y pegar es lento, propenso a errores y no escala: ¿y si el número de alienígenas depende del nivel? Lo que quieres decir es "repite esto 15 veces" o "repite esto mientras queden enemigos". Eso es un bucle.

-----

## Cómo funciona

### `while`: mientras la condición sea verdadera

```csharp
while (condicion)
{
    // se repite mientras condicion sea true
}
```

Se parece a un `if`, pero en lugar de ejecutar el bloque una vez, **vuelve a comprobar la condición** después de cada vuelta:

```csharp
int vidas = 3;

while (vidas > 0)
{
    Console.WriteLine($"Te quedan {vidas} vidas");
    vidas--;                      // sin esta línea, el bucle no terminaría nunca
}

Console.WriteLine("Fin del juego");
```

Salida:

```text
Te quedan 3 vidas
Te quedan 2 vidas
Te quedan 1 vidas
Fin del juego
```

Úsalo cuando sabes **cuándo parar**, pero no cuántas veces vas a repetir (leer hasta que el usuario escriba "salir", reintentar hasta que una conexión funcione).

Si la condición es `false` desde el principio, el bloque **no se ejecuta ni una vez**.

### `do...while`: al menos una vez

```csharp
do
{
    // se ejecuta una vez y luego se repite mientras condicion sea true
} while (condicion);   // ← lleva punto y coma
```

La condición se comprueba **después** de cada vuelta, así que el bloque se ejecuta siempre al menos una vez. Es ideal para menús y validación de entrada:

```csharp
int edad;
do
{
    Console.Write("Ingresa tu edad (0-120): ");
} while (!int.TryParse(Console.ReadLine(), out edad) || edad is < 0 or > 120);

Console.WriteLine($"Edad registrada: {edad}");
```

Primero se pide el dato y luego se decide si hay que volver a pedirlo.

### `for`: una cantidad conocida de veces

```csharp
for (inicialización; condición; actualización)
{
    // cuerpo
}
```

```csharp
for (int i = 0; i < 5; i++)
{
    Console.WriteLine($"Vuelta {i}");
}
// Vuelta 0, Vuelta 1, Vuelta 2, Vuelta 3, Vuelta 4
```

El orden exacto de ejecución:

1. **Inicialización** (`int i = 0`): se ejecuta **una sola vez**.
2. **Condición** (`i < 5`): si es `false`, el bucle termina.
3. **Cuerpo**.
4. **Actualización** (`i++`) y vuelta al paso 2.

`i` solo existe dentro del `for`. Variantes útiles:

```csharp
for (int i = 10; i > 0; i--) { }        // cuenta regresiva
for (int i = 0; i < 100; i += 5) { }    // de 5 en 5
for (int i = 0; i < arr.Length; i++) { } // recorrer un array por índice
```

Úsalo cuando sabes **cuántas veces** repetir o cuando necesitas el **índice**.

### `foreach`: por cada elemento

```csharp
foreach (tipo elemento in coleccion)
{
    // se ejecuta una vez por elemento
}
```

```csharp
string[] melodia = { "do", "re", "mi", "mi", "re" };

foreach (string nota in melodia)
{
    Console.WriteLine($"Tocando {nota}");
}
```

* Recorre en orden, del primero al último.
* No necesitas índices ni `Length`: no hay riesgo de pasarte del final.
* Funciona con cualquier colección: arrays, `List<T>`, strings (carácter por carácter), diccionarios...
* La variable de iteración es de **solo lectura**: `nota = "fa";` no compila (CS1656).

```csharp
foreach (char c in "Hola")
{
    Console.Write($"{c}-");      // H-o-l-a-
}
```

### Comparar los cuatro

El mismo recorrido escrito de tres formas:

```csharp
string[] items = { "poción", "daga", "escudo" };

// for: necesitas manejar el índice
for (int i = 0; i < items.Length; i++)
    Console.WriteLine(items[i]);

// while: manejas el índice y la actualización a mano
int j = 0;
while (j < items.Length)
{
    Console.WriteLine(items[j]);
    j++;
}

// foreach: la intención es clara y no hay índices que equivocar
foreach (string item in items)
    Console.WriteLine(item);
```

| Bucle | Úsalo cuando... |
| --- | --- |
| `while` | Conoces la condición de parada, no la cantidad de vueltas. |
| `do...while` | El bloque debe ejecutarse al menos una vez (menús, validación). |
| `for` | Conoces la cantidad de vueltas o necesitas el índice. |
| `foreach` | Quieres procesar cada elemento de una colección. Es la opción por defecto para recorrer. |

### `break`: salir del bucle

`break` termina el bucle de inmediato y continúa con lo que sigue después:

```csharp
int[] numeros = { 4, 8, 15, 16, 23, 42 };

foreach (int n in numeros)
{
    if (n > 15)
    {
        Console.WriteLine($"Primer número mayor a 15: {n}");
        break;                    // no hace falta revisar el resto
    }
}
```

Es la misma palabra que termina un caso de `switch`.

### `continue`: saltar a la siguiente vuelta

`continue` omite el resto del cuerpo **en esa vuelta** y pasa a la siguiente:

```csharp
for (int i = 0; i <= 10; i++)
{
    if (i < 9)
    {
        continue;                 // salta el WriteLine mientras i sea menor que 9
    }
    Console.WriteLine(i);
}
// Imprime 9 y 10
```

Uso típico: descartar los elementos que no interesan al principio del cuerpo.

```csharp
foreach (string linea in lineas)
{
    if (string.IsNullOrWhiteSpace(linea)) continue;   // ignorar líneas vacías
    Procesar(linea);
}
```

### `return`: salir del método entero

Dentro de un método, `return` termina **todo el método**, sin importar en cuántos bucles estés:

```csharp
static int BuscarPosicion(int[] datos, int buscado)
{
    for (int i = 0; i < datos.Length; i++)
    {
        if (datos[i] == buscado)
        {
            return i;             // sale del for Y del método
        }
    }
    return -1;                    // solo llega aquí si no lo encontró
}
```

* `break` → sale del **bucle** actual.
* `return` → sale del **método**.

### Bucles anidados

Un bucle dentro de otro. El interno se completa entero en cada vuelta del externo:

```csharp
for (int fila = 1; fila <= 3; fila++)
{
    for (int col = 1; col <= 3; col++)
    {
        Console.Write($"{fila * col,4}");
    }
    Console.WriteLine();
}
```

```text
   1   2   3
   2   4   6
   3   6   9
```

`break` solo sale del bucle **más interno**. Para salir de los dos, extrae el código a un método y usa `return`.

### Bucles infinitos

```csharp
while (true)
{
    Console.Write("Comando (salir para terminar): ");
    string? comando = Console.ReadLine();

    if (comando == "salir") break;   // la única salida
    Console.WriteLine($"Ejecutando {comando}");
}
```

Un bucle infinito intencional (`while (true)` + `break`) es válido cuando la condición de salida se conoce en mitad del cuerpo. Uno **accidental**, porque olvidaste actualizar la variable, congela el programa; detenlo con **Ctrl + C** en la terminal.

-----

## Ejemplo completo

Menú de un cajero:

```csharp
decimal saldo = 500m;
int opcion;

do
{
    Console.WriteLine();
    Console.WriteLine("1. Ver saldo");
    Console.WriteLine("2. Depositar");
    Console.WriteLine("3. Retirar");
    Console.WriteLine("4. Ver billetes para un monto");
    Console.WriteLine("0. Salir");
    Console.Write("Opción: ");

    if (!int.TryParse(Console.ReadLine(), out opcion))
    {
        Console.WriteLine("Opción inválida.");
        opcion = -1;
        continue;                       // vuelve a evaluar la condición del do...while
    }

    switch (opcion)
    {
        case 1:
            Console.WriteLine($"Saldo: {saldo:N2}");
            break;
        case 2:
            Console.Write("Monto: ");
            if (decimal.TryParse(Console.ReadLine(), out decimal deposito) && deposito > 0)
                saldo += deposito;
            break;
        case 3:
            Console.Write("Monto: ");
            if (decimal.TryParse(Console.ReadLine(), out decimal retiro) && retiro > 0 && retiro <= saldo)
                saldo -= retiro;
            else
                Console.WriteLine("Monto no permitido.");
            break;
        case 4:
            Console.Write("Monto entero: ");
            if (int.TryParse(Console.ReadLine(), out int monto) && monto > 0)
            {
                int[] billetes = { 200, 100, 50, 20, 10 };
                foreach (int b in billetes)
                {
                    int cantidad = monto / b;
                    if (cantidad == 0) continue;
                    Console.WriteLine($"{cantidad} x {b}");
                    monto %= b;
                }
                if (monto > 0) Console.WriteLine($"Resto sin billete: {monto}");
            }
            break;
    }
} while (opcion != 0);

Console.WriteLine("¡Hasta luego!");
```

Ejecución de ejemplo (opción 4 con 380):

```text
Opción: 4
Monto entero: 380
1 x 200
1 x 100
1 x 50
1 x 20
1 x 10
```

`do...while` mantiene el menú, `switch` elige la acción, `foreach` recorre los billetes y `continue` salta los que no aplican.

-----

## Errores comunes

**1. Error por uno (*off-by-one*).**
Qué pasa: `IndexOutOfRangeException` o un elemento de menos.
Por qué: `i <= arr.Length` llega a un índice que no existe; `i < arr.Length - 1` omite el último.
Arreglo: el patrón estándar es `for (int i = 0; i < arr.Length; i++)`, o directamente `foreach`.

**2. Olvidar actualizar la variable de un `while`.**
Qué pasa: bucle infinito; el programa se congela.
Por qué: la condición nunca cambia.
Arreglo: asegúrate de que algo en el cuerpo acerque la condición a `false`.

**3. Punto y coma después del `for` o del `while`.**
Qué pasa: `for (int i = 0; i < 5; i++);` compila con `warning CS0642` y el bloque se ejecuta **una sola vez**.
Por qué: el `;` es un cuerpo vacío y las llaves de abajo quedan fuera del bucle.
Arreglo: quita el `;`.

**4. Olvidar el `;` del `do...while`.**
Qué pasa: `error CS1002: ; expected`.
Por qué: `do...while` es la única estructura de bucle que termina en punto y coma.
Arreglo: `} while (condicion);`.

**5. Asignar a la variable del `foreach`.**
Qué pasa: `error CS1656: Cannot assign to 'nota' because it is a 'foreach iteration variable'`.
Por qué: la variable de iteración es de solo lectura.
Arreglo: usa `for` con índice si necesitas reemplazar elementos del array.

**6. Modificar una lista mientras la recorres con `foreach`.**
Qué pasa: `System.InvalidOperationException: Collection was modified; enumeration operation may not execute.`
Por qué: `foreach` no admite cambios en la colección (agregar o quitar) durante el recorrido.
Arreglo: recorre una copia (`foreach (var x in lista.ToList())`), usa un `for` hacia atrás o `lista.RemoveAll(...)`.

-----

## Según la versión de C#

* **C# 5:** la variable de `foreach` pasó a ser **nueva en cada vuelta**. Antes, las lambdas creadas dentro de un `foreach` capturaban todas la misma variable (un bug clásico). En un `for`, la variable sigue siendo una sola para todo el bucle (ver "Para profundizar").
* **C# 8:** `await foreach` para recorrer flujos asíncronos (`IAsyncEnumerable<T>`).
* **C# 8 / C# 12:** índices, rangos y expresiones de colección, que reducen muchos bucles manuales de copia.

-----

## Cuándo sí y cuándo no

**Prefiere `foreach` cuando:**

* Solo necesitas cada elemento. Es más legible y no tiene errores por uno.

**Prefiere `for` cuando:**

* Necesitas el índice, recorrer al revés, saltar de a varios o modificar elementos del array.

**Considera LINQ en lugar de un bucle cuando:**

* El bucle solo filtra, transforma o suma (`numeros.Where(...).Sum()`). Para lógica con varios pasos y efectos (imprimir, guardar), un bucle suele ser más claro.

**Evita:**

* Más de dos niveles de bucles anidados: extrae métodos.
* `break` y `continue` repartidos por todo un cuerpo largo: dificultan seguir el flujo.

-----

## Resumen en 5 líneas

1. `while` repite mientras la condición sea `true`; `do...while` lo hace al menos una vez.
2. `for (init; condición; actualización)` sirve para cantidades conocidas o cuando necesitas el índice.
3. `foreach (var x in coleccion)` recorre cada elemento; es la opción por defecto para colecciones.
4. `break` sale del bucle, `continue` salta a la siguiente vuelta y `return` sale del método.
5. Cuidado con los errores por uno (`<` frente a `<=`) y con modificar una colección dentro de un `foreach`.

-----

## Para profundizar

<details>
<summary>Lambdas dentro de un for: la variable compartida</summary>

```csharp
var acciones = new List<Action>();
for (int i = 0; i < 3; i++)
{
    acciones.Add(() => Console.Write(i));
}
foreach (var a in acciones) a();   // imprime 333, no 012
```

Las tres lambdas capturan **la misma** variable `i`, que al final vale 3. Solución: copiarla a una variable local dentro del cuerpo (`int copia = i;`) y capturar `copia`. Con `foreach` esto ya no pasa desde C# 5.

</details>

<details>
<summary>Cómo funciona foreach por dentro</summary>

`foreach` no es magia: el compilador lo traduce a llamadas a `GetEnumerator()`, `MoveNext()` y `Current`:

```csharp
var e = coleccion.GetEnumerator();
while (e.MoveNext())
{
    var item = e.Current;
    // cuerpo
}
```

Cualquier tipo que tenga ese método `GetEnumerator()` se puede recorrer con `foreach`. Con arrays, el compilador lo optimiza como un `for` por índice. Se retoma en [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

`while` repite mientras una condición sea verdadera; `do...while` igual, pero se ejecuta al menos una vez; `for` se usa cuando sabes cuántas veces repetir o necesitas el índice; y `foreach` recorre cada elemento de una colección. `break` sale del bucle y `continue` salta a la siguiente iteración.

### Respuesta ampliada (semi-senior)

`foreach` trabaja sobre el patrón `GetEnumerator`/`MoveNext`/`Current` (o `IEnumerable<T>`); con arrays se compila como un `for` sin enumerador. Modificar una `List<T>` durante un `foreach` invalida el enumerador (`InvalidOperationException`) porque la lista lleva una versión interna. Desde C# 5, la variable de `foreach` es nueva en cada iteración, lo que evita el bug de captura en clausuras, que sigue existiendo en `for`. Para salir de bucles anidados se prefiere extraer un método y usar `return`. Muchos bucles de filtrado y agregación se expresan mejor con LINQ, aunque en rutas críticas de rendimiento un bucle simple evita las asignaciones de los delegados.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `while` y `do...while`?**
`while` comprueba antes de ejecutar (puede no ejecutar nunca); `do...while` comprueba después (ejecuta al menos una vez).

**2. ¿Se puede modificar una lista dentro de un `foreach`?**
No agregar ni quitar elementos: lanza `InvalidOperationException`. Sí modificar propiedades de los objetos que contiene.

**3. ¿`break` sale de todos los bucles anidados?**
No, solo del más interno.

-----

## Práctica

**Ejercicio 1.** Escribe un programa que imprima los números del 1 al 30, pero que reemplace los múltiplos de 3 por `Fizz`, los de 5 por `Buzz` y los de ambos por `FizzBuzz`.

<details>
<summary>Solución</summary>

```csharp
for (int i = 1; i <= 30; i++)
{
    string salida = (i % 3, i % 5) switch
    {
        (0, 0) => "FizzBuzz",
        (0, _) => "Fizz",
        (_, 0) => "Buzz",
        _ => i.ToString()
    };
    Console.WriteLine(salida);
}
```

También es válida la versión con `if / else if`, comprobando primero `i % 15 == 0`. Lo importante es el orden: el caso de "ambos" va primero.

</details>

**Ejercicio 2.** Pide números al usuario hasta que escriba `0`. Ignora (con `continue`) los textos que no sean números y, al final, muestra la suma y el máximo de los números ingresados.

<details>
<summary>Solución</summary>

```csharp
int suma = 0;
int? maximo = null;

while (true)
{
    Console.Write("Número (0 para terminar): ");
    if (!int.TryParse(Console.ReadLine(), out int n))
    {
        Console.WriteLine("No es un número, se ignora.");
        continue;
    }

    if (n == 0) break;

    suma += n;
    if (maximo is null || n > maximo) maximo = n;
}

Console.WriteLine($"Suma: {suma}");
Console.WriteLine($"Máximo: {maximo?.ToString() ?? "sin datos"}");
```

`int?` permite representar "todavía no hay máximo", sin inventar un valor inicial arbitrario. Un ternario `maximo is null ? "sin datos" : maximo` no compilaría (CS0173), porque sus ramas serían `string` e `int?`.

</details>

-----

## Siguiente lección

Terminaste el módulo de control de flujo. Continúa con [Métodos](../03-metodos/README.md).
