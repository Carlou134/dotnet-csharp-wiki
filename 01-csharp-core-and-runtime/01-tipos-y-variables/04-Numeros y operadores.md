# Números y operadores

## En una frase

C# tiene tipos para enteros (`int`, `long`) y para decimales (`double`, `decimal`, `float`), operadores aritméticos (`+ - * / %`) cuyo resultado **depende del tipo de los operandos**, y la clase `Math` para cálculos más avanzados.

-----

## Antes de empezar

Conviene que ya sepas:

* Declarar variables numéricas y la tabla de tipos, de [Variables y tipos de datos](01-Variables%20y%20tipos%20de%20datos.md).
* Qué es una conversión implícita y un cast, de [Conversiones de tipos](03-Conversiones%20de%20tipos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Operador:** símbolo que realiza una operación sobre uno o más valores (*operandos*).
* **Operador unario / binario:** actúa sobre un operando (`-x`, `x++`) o sobre dos (`a + b`).
* **Precedencia:** el orden en que se evalúan los operadores en una expresión.
* **División entera:** la división entre dos enteros, que descarta la parte decimal.
* **Módulo (resto):** lo que sobra de una división entera, con el operador `%`.
* **Punto flotante:** representación binaria aproximada de números con decimales (`float`, `double`).
* **Asignación compuesta:** abreviaturas como `+=` o `*=`.

-----

## El problema

Esta línea parece obvia:

```csharp
Console.WriteLine(5 / 2);
```

Imprime `2`, no `2.5`. Y esta otra:

```csharp
Console.WriteLine(0.1 + 0.2 == 0.3);
```

Imprime `False`. Si calculas notas, precios o estadísticas sin entender **por qué**, tendrás resultados incorrectos que nadie detecta hasta que un cliente reclama. Los números en programación no se comportan exactamente como en la calculadora.

-----

## Cómo funciona

### Elegir el tipo numérico

| Pregunta | Tipo |
| --- | --- |
| ¿Es una cantidad entera (personas, intentos, índices)? | `int` |
| ¿El entero puede superar ~2.100 millones (IDs grandes, milisegundos)? | `long` |
| ¿Es una medida con decimales (distancias, ciencia, gráficos)? | `double` |
| ¿Es dinero o necesitas decimales exactos? | `decimal` |
| ¿Memoria muy limitada o APIs gráficas que lo exigen? | `float` |

```csharp
int intentos = 3;
long milisegundos = 1_728_000_000_000;   // _ separa grupos de dígitos (no cambia el valor)
double distanciaKm = 384_400.5;
decimal precio = 489_872.76m;            // sufijo m obligatorio
float escala = 1.5f;                     // sufijo f obligatorio
```

### Operadores aritméticos

| Operador | Operación | Ejemplo | Resultado |
| --- | --- | --- | --- |
| `+` | Suma | `7 + 2` | `9` |
| `-` | Resta | `7 - 2` | `5` |
| `*` | Multiplicación | `7 * 2` | `14` |
| `/` | División | `7 / 2` | `3` (¡entera!) |
| `%` | Módulo (resto) | `7 % 2` | `1` |

### División entera: el tipo manda

Si **los dos** operandos son enteros, el resultado es entero y se descarta la parte decimal:

```csharp
Console.WriteLine(5 / 2);       // 2
Console.WriteLine(5 / 2.0);     // 2.5 (un operando double → resultado double)
Console.WriteLine(5.0 / 2);     // 2.5
Console.WriteLine((double)5 / 2); // 2.5
```

Un error clásico es calcular un promedio así:

```csharp
int suma = 17;
int cantidad = 4;
double promedio = suma / cantidad;     // 4 ← la división ya fue entera ANTES de asignarse
double promedioOk = (double)suma / cantidad;   // 4.25
```

Asignar el resultado a un `double` no arregla nada: la operación ya se hizo con enteros.

### Módulo `%`

Devuelve el resto de la división entera:

```csharp
Console.WriteLine(10 % 3);    // 1
Console.WriteLine(12 % 4);    // 0
Console.WriteLine(-7 % 3);    // -1 (el signo es el del dividendo)
```

Usos típicos:

```csharp
bool esPar = numero % 2 == 0;

int huevos = 56;
int porCaja = 12;
int cajasLlenas = huevos / porCaja;   // 4
int sobrantes = huevos % porCaja;     // 8
```

### Precedencia de operadores

```csharp
Console.WriteLine(1 + 2 * 3);      // 7, no 9
Console.WriteLine((1 + 2) * 3);    // 9
```

Orden, de mayor a menor prioridad (resumido):

1. Paréntesis `( )`
2. Unarios: `++x`, `--x`, `-x`, `!x`, casts
3. Multiplicativos: `*`, `/`, `%`
4. Aditivos: `+`, `-`
5. Comparación: `<`, `>`, `<=`, `>=`, luego `==`, `!=`
6. Lógicos: `&&`, luego `||`
7. Asignación: `=`, `+=`...

Con la misma precedencia, se evalúa de izquierda a derecha: `10 - 4 - 3` es `3`.

C# **no tiene operador de potencia**: `2 ^ 3` es un XOR de bits (da `1`), no 2 al cubo. Usa `Math.Pow(2, 3)`.

Ante la duda, **usa paréntesis**. Hacen explícita la intención y nadie tiene que recordar la tabla.

### Asignación compuesta

```csharp
int puntos = 10;
puntos += 5;    // puntos = puntos + 5  → 15
puntos -= 3;    // 12
puntos *= 2;    // 24
puntos /= 4;    // 6
puntos %= 4;    // 2
```

### Incremento y decremento: `++` y `--`

Suman o restan 1. La **posición** importa cuando usas el resultado en la misma expresión:

```csharp
int x = 5;
int a = x++;   // POST-incremento: a = 5, después x = 6
int b = ++x;   // PRE-incremento: x = 7, después b = 7
```

| Forma | Qué hace | Devuelve |
| --- | --- | --- |
| `x++` / `x--` | Usa el valor y **después** lo cambia | El valor anterior |
| `++x` / `--x` | Cambia el valor y **después** lo usa | El valor nuevo |

```csharp
int n = 3;
Console.WriteLine(n++);   // 3 (luego n = 4)
Console.WriteLine(++n);   // 5
```

Como sentencia sola (`contador++;`) las dos formas son equivalentes. Evita mezclarlas dentro de expresiones largas: `y = x++ + ++x;` compila, pero nadie debería tener que descifrarlo.

### La clase `Math`

```csharp
Math.Abs(-5);            // 5        valor absoluto
Math.Sqrt(16);           // 4        raíz cuadrada
Math.Pow(2, 10);         // 1024     potencia (devuelve double)
Math.Max(39, 12);        // 39
Math.Min(39, 12);        // 12
Math.Floor(8.65);        // 8        redondea hacia abajo
Math.Ceiling(8.15);      // 9        redondea hacia arriba
Math.Round(8.65m, 1);    // 8.6      redondeo bancario (ver "Para profundizar")
Math.Truncate(-8.65);    // -8       corta los decimales
Math.Clamp(150, 0, 100); // 100      limita a un rango
Math.PI;                 // 3.141592653589793
```

`Math.Sqrt(-1)` **no** lanza un error: devuelve `NaN` (*Not a Number*). Compruébalo con `double.IsNaN(resultado)`.

### Límites de los tipos

```csharp
Console.WriteLine(int.MaxValue);    // 2147483647
Console.WriteLine(int.MinValue);    // -2147483648

int max = int.MaxValue;
max++;
Console.WriteLine(max);             // -2147483648 (desbordamiento: da la vuelta)
```

En contexto `checked` ese `++` lanzaría `OverflowException` (ver [Conversiones de tipos](03-Conversiones%20de%20tipos.md)).

### Dividir por cero

```csharp
int a = 10, b = 0;
Console.WriteLine(a / b);       // System.DivideByZeroException

double c = 10, d = 0;
Console.WriteLine(c / d);       // ∞ (double.PositiveInfinity), no lanza excepción
Console.WriteLine(0.0 / 0.0);   // NaN
```

Los enteros fallan; los `double` siguen la norma IEEE 754 y devuelven infinito o `NaN`.

### Números aleatorios

```csharp
int dado = Random.Shared.Next(1, 7);      // entre 1 y 6: el límite superior NO se incluye
double proba = Random.Shared.NextDouble(); // entre 0.0 y 1.0 (sin incluir el 1.0)

var random = new Random();                 // también puedes crear tu instancia
int numero = random.Next(100);             // entre 0 y 99
```

### Conocer el tipo de un valor

```csharp
var x = 5 / 2.0;
Console.WriteLine(x.GetType());      // System.Double (tipo del valor al ejecutar)
Console.WriteLine(typeof(int));      // System.Int32 (tipo nombrado al compilar)
```

-----

## Ejemplo completo

```csharp
int[] notas = { 15, 18, 12, 17 };

int suma = 0;
foreach (int nota in notas)
{
    suma += nota;
}

double promedio = (double)suma / notas.Length;
double promedioRedondeado = Math.Round(promedio, 2);
bool aprobado = promedio >= 13;
int notaMaxima = Math.Max(Math.Max(notas[0], notas[1]), Math.Max(notas[2], notas[3]));

Console.WriteLine($"Suma: {suma}");
Console.WriteLine($"Promedio: {promedioRedondeado}");
Console.WriteLine($"Aprobado: {aprobado}");
Console.WriteLine($"Nota máxima: {notaMaxima}");
Console.WriteLine($"¿Suma par?: {suma % 2 == 0}");
```

Salida:

```text
Suma: 62
Promedio: 15.5
Aprobado: True
Nota máxima: 18
¿Suma par?: True
```

El `foreach` y el array se estudian en [Arrays](../02-control-de-flujo/03-Arrays.md) y [Bucles](../02-control-de-flujo/04-Bucles.md). Lo importante aquí es el `(double)suma`: sin él, el promedio sería `15`.

-----

## Errores comunes

**1. División entera cuando esperabas decimales.**
Qué pasa: `7 / 2` da `3` y los porcentajes o promedios salen mal, sin ningún error.
Por qué: dos enteros producen un entero.
Arreglo: convierte un operando antes de dividir: `(double)a / b` o `a / 2.0`.

**2. Usar `double` para dinero.**
Qué pasa: `0.1 + 0.2` da `0.30000000000000004` y las sumas de una factura no cuadran.
Por qué: `double` usa base 2 y no puede representar 0.1 exactamente.
Arreglo: `decimal` para dinero: `0.1m + 0.2m == 0.3m` es `true`.

**3. Comparar `double` con `==`.**
Qué pasa: `0.1 + 0.2 == 0.3` da `false`.
Por qué: errores de redondeo binario.
Arreglo: compara con una tolerancia: `Math.Abs(a - b) < 1e-9`.

**4. Usar `^` como potencia.**
Qué pasa: `2 ^ 3` da `1` sin ningún error.
Por qué: `^` es el XOR (o exclusivo) de bits.
Arreglo: `Math.Pow(2, 3)`.

**5. Creer que `Random.Next(1, 6)` incluye el 6.**
Qué pasa: el dado nunca saca 6.
Por qué: el segundo argumento es exclusivo.
Arreglo: `Next(1, 7)`.

**6. Olvidar paréntesis en `new Random`.**
Qué pasa: `error CS1526: A new expression requires an argument list or (), [], or {} after type`.
Por qué: `new Random;` no es una llamada válida al constructor.
Arreglo: `new Random()`.

**7. Dividir enteros por cero.**
Qué pasa: `System.DivideByZeroException`.
Por qué: la división entera por cero no está definida.
Arreglo: verifica el divisor antes de dividir.

-----

## Según la versión de C#

* **C# 7:** separador de dígitos `_` y literales binarios (`0b1111_0000`).
* **.NET 6:** `Random.Shared`, una instancia compartida y segura entre hilos.
* **.NET 7 (C# 11):** *generic math* (`INumber<T>`): métodos genéricos que funcionan con cualquier tipo numérico, y el tipo `Int128`.
* **C# 11:** operador de desplazamiento sin signo `>>>`.

-----

## Cuándo sí y cuándo no

**Usa `decimal` cuando:**

* Manejas dinero, impuestos, tasas o cualquier valor donde un centavo de diferencia importa.

**Usa `double` cuando:**

* Calculas física, estadística, gráficos o mediciones donde una aproximación mínima es aceptable. Es mucho más rápido que `decimal`.

**Usa `++` / `--` cuando:**

* Incrementas un contador como sentencia sola. Evítalos dentro de expresiones complejas.

-----

## Resumen en 5 líneas

1. `int` para enteros, `double` para decimales generales, `decimal` para dinero.
2. Entero dividido por entero da entero: `5 / 2` es `2`; usa `(double)` antes de dividir.
3. `%` da el resto; `++x` cambia antes de usar y `x++` usa antes de cambiar.
4. No hay operador de potencia: `Math.Pow`. Usa paréntesis para dejar clara la precedencia.
5. `double` no es exacto (`0.1 + 0.2 != 0.3`); `Random.Next(min, max)` excluye `max`.

-----

## Para profundizar

<details>
<summary>Por qué 0.1 + 0.2 no da 0.3</summary>

`double` guarda los números en binario. Igual que 1/3 no se puede escribir exactamente en decimal (0.3333...), 0.1 no se puede escribir exactamente en binario: es una fracción periódica. El valor guardado es el más cercano posible, y al sumar dos aproximaciones el error se nota:

```csharp
Console.WriteLine(0.1 + 0.2);            // 0.30000000000000004
Console.WriteLine(0.1m + 0.2m);          // 0.3
```

`decimal` guarda un entero de 96 bits y un factor de escala en base 10, así que 0.1 es exacto. A cambio, es más lento y tiene menor rango.

</details>

<details>
<summary>Math.Round y el redondeo bancario</summary>

Por defecto, `Math.Round` usa `MidpointRounding.ToEven`: cuando el valor está exactamente en la mitad, redondea al par más cercano (`2.5` → `2`, `3.5` → `4`). Así se reduce el sesgo al sumar muchos redondeos. Si necesitas el redondeo escolar ("la mitad sube"), indícalo:

```csharp
Math.Round(2.5);                                   // 2
Math.Round(2.5, MidpointRounding.AwayFromZero);    // 3
```

Además, con `double` el "punto medio" casi nunca es exacto: 8.65 no se puede guardar tal cual en binario, sino como un valor apenas por encima o por debajo, así que `Math.Round(8.65, 1)` no aplica la regla del punto medio como esperarías. Con `decimal` (`8.65m`) el valor es exacto y el resultado es predecible: `8.6`.

</details>

<details>
<summary>Random no sirve para seguridad</summary>

`Random` genera números **pseudoaleatorios**, predecibles si se conoce el estado interno. Para tokens, contraseñas o claves usa `System.Security.Cryptography.RandomNumberGenerator`:

```csharp
int codigo = RandomNumberGenerator.GetInt32(100_000, 1_000_000);
```

</details>

-----

## En entrevista

### Respuesta corta (junior)

La división entre dos enteros es entera: `5 / 2` da 2. Para obtener decimales, uno de los operandos tiene que ser `double` o `decimal`. Para dinero se usa `decimal` porque `double` tiene errores de precisión. El `%` da el resto, y `x++` usa el valor antes de incrementarlo, mientras que `++x` lo incrementa primero.

### Respuesta ampliada (semi-senior)

El tipo del resultado lo determinan los operandos, no la variable donde se asigna, así que `double r = a / b;` con enteros ya pierde los decimales. `double` es IEEE 754 binario, rápido pero inexacto para fracciones decimales; `decimal` es base 10 con 28-29 dígitos significativos, adecuado para dinero. La aritmética entera es `unchecked` por defecto y desborda en silencio; la división entera por cero lanza `DivideByZeroException`, mientras que en `double` produce infinito o `NaN`. `Math.Round` usa redondeo bancario por defecto. Para aleatoriedad en general se usa `Random.Shared`, y para seguridad, `RandomNumberGenerator`.

### Preguntas frecuentes de seguimiento

**1. ¿Qué imprime `Console.WriteLine(7 / 2 * 2.0);`?**
`6`: primero `7 / 2` es la división entera `3`, y luego `3 * 2.0` es `6.0`.

**2. ¿Cuál es la diferencia entre `x++` y `++x`?**
Los dos suman 1; `x++` devuelve el valor anterior y `++x` el nuevo.

**3. ¿Por qué `decimal` y no `double` para dinero?**
Porque representa las fracciones decimales exactamente; `double` acumula errores binarios.

**4. ¿Qué pasa al sumar 1 a `int.MaxValue`?**
En contexto `unchecked` da `int.MinValue`; en `checked` lanza `OverflowException`.

-----

## Práctica

**Ejercicio 1.** Sin ejecutar, ¿qué imprime?

```csharp
int a = 10;
int b = 4;
Console.WriteLine(a / b);
Console.WriteLine(a % b);
Console.WriteLine((double)a / b);
Console.WriteLine(a / b * 1.0);
int c = a++ + 1;
Console.WriteLine($"{a} {c}");
```

<details>
<summary>Solución</summary>

```text
2
2
2.5
2
11 11
```

`a / b * 1.0`: primero `a / b` (entera) = 2, luego 2 × 1.0 = 2. En `c = a++ + 1`, se usa `a` = 10 (c = 11) y después `a` pasa a 11.

</details>

**Ejercicio 2.** Escribe un programa que convierta una cantidad de segundos (por ejemplo, `3725`) en horas, minutos y segundos usando `/` y `%`.

<details>
<summary>Solución</summary>

```csharp
int totalSegundos = 3725;

int horas = totalSegundos / 3600;
int minutos = totalSegundos % 3600 / 60;
int segundos = totalSegundos % 60;

Console.WriteLine($"{horas}h {minutos}m {segundos}s");   // 1h 2m 5s
```

</details>

-----

## Siguiente lección

[Texto: char y string](05-Texto%20char%20y%20string.md)
