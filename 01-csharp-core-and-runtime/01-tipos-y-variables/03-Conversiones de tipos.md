# Conversiones de tipos

## En una frase

Convertir es pasar un valor de un tipo a otro: C# lo hace **solo** cuando no se pierde información (implícita), te exige un **cast** cuando puede perderse (explícita) y, para convertir **texto** en números, ofrece `Parse`, `TryParse` y `Convert`.

-----

## Antes de empezar

Conviene que ya sepas:

* Los tipos numéricos y sus tamaños, de [Variables y tipos de datos](01-Variables%20y%20tipos%20de%20datos.md).
* Que `Console.ReadLine()` siempre devuelve `string`, como en [Tu primer programa](../00-introduccion/03-Tu%20primer%20programa.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Conversión implícita (ampliación, *widening*):** automática, porque el tipo destino puede representar todos los valores del origen.
* **Conversión explícita (restricción, *narrowing*):** requiere escribir el tipo destino entre paréntesis, porque puede perder datos.
* **Cast:** la sintaxis `(tipo)valor` que pide una conversión explícita.
* **Truncar:** cortar la parte decimal sin redondear.
* **Parsear:** interpretar un texto como otro tipo (`"42"` → `42`).
* **Excepción:** un error en tiempo de ejecución que, si nadie lo maneja, detiene el programa.
* **Desbordamiento (*overflow*):** cuando un valor no cabe en el tipo destino.

-----

## El problema

Los datos casi nunca llegan con el tipo que necesitas:

* El usuario escribe `"25"` en la consola: es texto, pero quieres calcular con el número.
* Tienes un promedio `double` (`7.8`) y necesitas guardarlo en un `int`.
* Tienes un `int` y una función pide un `long`.

Algunas de estas conversiones son seguras y otras pueden perder datos o fallar. C# distingue cada caso para que **la pérdida de datos nunca ocurra en silencio**.

-----

## Cómo funciona

### Conversión implícita: no se pierde nada

Si el tipo destino puede representar **todos** los valores del origen, C# convierte solo:

```csharp
int entero = 42;
long grande = entero;       // int → long: siempre cabe
double real = entero;       // int → double: 42.0
float f = 10;               // int → float
```

Caminos implícitos habituales:

```text
byte → short → int → long → float → double
              char → int
int, long → decimal
```

### Conversión explícita (cast): puede perderse algo

Si puede perderse información, el compilador te obliga a decir "lo sé, hazlo igual":

```csharp
double precio = 9.99;
int precioEntero = precio;          // error CS0266: falta una conversión explícita
int precioEntero2 = (int)precio;    // 9 → TRUNCA, no redondea
```

```csharp
Console.WriteLine((int)9.99);    // 9
Console.WriteLine((int)-9.99);   // -9 (trunca hacia cero)
Console.WriteLine((int)'A');     // 65 (código del carácter)
Console.WriteLine((char)66);     // B
```

Si quieres **redondear**, usa `Math.Round` antes del cast:

```csharp
int redondeado = (int)Math.Round(9.99);   // 10
```

### Desbordamiento: cuando el valor no cabe

```csharp
int grande = 300;
byte chico = (byte)grande;
Console.WriteLine(chico);    // 44 (!)
```

`byte` solo llega hasta 255. Por defecto, C# **no** da error: se queda con los bits que caben (300 − 256 = 44). Para que falle en lugar de dar un valor incorrecto, usa `checked`:

```csharp
byte seguro = checked((byte)grande);   // System.OverflowException
```

### De texto a número: `Parse`

```csharp
string texto = "123";
int numero = int.Parse(texto);           // 123
double d = double.Parse("3.5");          // 3.5 (depende de la cultura, ver abajo)
bool b = bool.Parse("true");             // true
```

`Parse` **lanza una excepción** si el texto no es válido:

```csharp
int.Parse("hola");     // System.FormatException
int.Parse("");         // System.FormatException
int.Parse(null);       // System.ArgumentNullException
int.Parse("99999999999"); // System.OverflowException
```

Úsalo solo cuando estés **seguro** de que el texto es válido (por ejemplo, un dato que tú mismo generaste).

### De texto a número de forma segura: `TryParse`

`TryParse` **intenta** convertir y te dice si pudo, sin lanzar excepciones:

```csharp
string entrada = Console.ReadLine() ?? "";

if (int.TryParse(entrada, out int edad))
{
    Console.WriteLine($"El año que viene tendrás {edad + 1}.");
}
else
{
    Console.WriteLine("Eso no es un número válido.");
}
```

* Devuelve `true` si la conversión funcionó y deja el resultado en la variable marcada con `out`.
* Devuelve `false` si falló y deja la variable en su valor por defecto (`0` para `int`).
* `out int edad` declara la variable en la misma llamada. La palabra `out` se explica en [Valores de retorno y parámetros out](../03-metodos/03-Valores%20de%20retorno%20y%20parametros%20out.md).

Existe en todos los tipos básicos: `double.TryParse`, `decimal.TryParse`, `bool.TryParse`, `DateTime.TryParse`...

**Regla práctica:** para datos que vienen del usuario, de archivos o de la red, usa siempre `TryParse`.

### La clase `Convert`

`Convert` tiene métodos `ToX` para casi todas las combinaciones:

```csharp
int a = Convert.ToInt32("42");       // 42
double b = Convert.ToDouble(10);     // 10
string c = Convert.ToString(3.5);    // "3.5"
bool d = Convert.ToBoolean(1);       // true
```

Diferencias importantes con el cast y con `Parse`:

```csharp
Console.WriteLine((int)2.5);                 // 2  → el cast TRUNCA
Console.WriteLine(Convert.ToInt32(2.5));     // 2  → Convert REDONDEA... al par más cercano
Console.WriteLine(Convert.ToInt32(3.5));     // 4
Console.WriteLine(Convert.ToInt32(null));    // 0  → no lanza excepción con null
// int.Parse(null) lanzaría ArgumentNullException
```

`Convert.ToInt32(2.5)` da `2` y no `3` porque usa el **redondeo bancario**: cuando el valor está justo en la mitad, redondea al número **par** más cercano. Es lo mismo que hace `Math.Round` por defecto.

### Cualquier cosa a texto: `ToString`

Todo valor tiene `ToString()`:

```csharp
int n = 42;
string s = n.ToString();               // "42"
string m = 1234.5m.ToString("C");      // "$1,234.50" (o "S/ 1,234.50", según la cultura)
string p = 0.256.ToString("P1");       // "25.6%" (el formato exacto varía según la cultura)
```

### La cultura importa

Los números con decimales y las fechas se escriben distinto según el país: `3.5` en EE. UU., `3,5` en España o Argentina. `Parse`, `TryParse` y `ToString` usan la **cultura del sistema** si no indicas otra:

```csharp
using System.Globalization;

// En una máquina configurada en español, esto puede FALLAR o dar 35:
double x = double.Parse("3.5");

// Independiente de la configuración:
double y = double.Parse("3.5", CultureInfo.InvariantCulture);   // 3.5 siempre
```

Usa `CultureInfo.InvariantCulture` para datos que leen o escriben **programas** (JSON, CSV, configuración) y la cultura del usuario para lo que **lee una persona**.

-----

## Ejemplo completo

```csharp
using System.Globalization;

Console.Write("Precio del producto: ");
string textoPrecio = Console.ReadLine() ?? "";

Console.Write("Cantidad: ");
string textoCantidad = Console.ReadLine() ?? "";

bool precioOk = decimal.TryParse(textoPrecio, NumberStyles.Number, CultureInfo.InvariantCulture, out decimal precio);
bool cantidadOk = int.TryParse(textoCantidad, out int cantidad);

if (!precioOk || !cantidadOk)
{
    Console.WriteLine("Datos inválidos. Usa punto para los decimales (por ejemplo 19.90).");
    return;
}

decimal total = precio * cantidad;           // int se convierte implícitamente a decimal
int totalEntero = (int)total;                // cast explícito: trunca
int totalRedondeado = (int)Math.Round(total);

Console.WriteLine($"Total exacto: {total}");
Console.WriteLine($"Total truncado: {totalEntero}");
Console.WriteLine($"Total redondeado: {totalRedondeado}");
```

Ejecución de ejemplo:

```text
Precio del producto: 19.90
Cantidad: 3
Total exacto: 59.70
Total truncado: 59
Total redondeado: 60
```

-----

## Errores comunes

**1. Asignar un `double` a un `int` sin cast.**
Qué pasa: `error CS0266: Cannot implicitly convert type 'double' to 'int'. An explicit conversion exists (are you missing a cast?)`.
Por qué: perderías los decimales.
Arreglo: `(int)valor` si quieres truncar o `(int)Math.Round(valor)` si quieres redondear.

**2. Asignar un `string` a un número.**
Qué pasa: `error CS0029: Cannot implicitly convert type 'string' to 'int'`.
Por qué: un texto no es un número; no existe conversión implícita ni cast entre ellos.
Arreglo: `int.TryParse(texto, out int n)`.

**3. Usar `Parse` con datos del usuario.**
Qué pasa: `System.FormatException: The input string 'abc' was not in a correct format.` y el programa se detiene.
Por qué: el usuario puede escribir cualquier cosa.
Arreglo: `TryParse` y manejar el caso `false`.

**4. Creer que el cast redondea.**
Qué pasa: `(int)2.99` da `2` y los cálculos salen mal.
Por qué: el cast trunca hacia cero.
Arreglo: `Math.Round` antes del cast, eligiendo el modo de redondeo si importa (`MidpointRounding.AwayFromZero`).

**5. Ignorar la cultura al parsear decimales.**
Qué pasa: `"3.5"` se lee como `35` o falla en una máquina en español.
Por qué: en esa cultura el punto es separador de miles y la coma es el separador decimal.
Arreglo: `CultureInfo.InvariantCulture` para datos de máquina.

**6. Desbordamiento silencioso.**
Qué pasa: `(byte)300` da `44` y nadie se entera.
Por qué: por defecto, las conversiones numéricas no verifican el desbordamiento.
Arreglo: `checked(...)` o activar `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` en el `.csproj`.

-----

## Según la versión de C#

* **C# 7:** `out var` en la misma llamada: `int.TryParse(s, out int n)`. Antes había que declarar `int n;` en una línea previa.
* **.NET Core 2.1+:** sobrecargas de `Parse`/`TryParse` que aceptan `ReadOnlySpan<char>`, para parsear sin crear strings intermedios.
* **.NET 7 (C# 11):** interfaces `IParsable<T>` y `INumber<T>`, que permiten escribir métodos genéricos de parseo y aritmética (*generic math*).

-----

## Cuándo sí y cuándo no

| Necesitas... | Usa | Evita |
| --- | --- | --- |
| Ampliar un número (`int` → `long`) | Asignación implícita | Casts innecesarios |
| Reducir un número sabiendo que puede perder datos | `(tipo)valor` | Hacerlo sin pensar en el desbordamiento |
| Convertir texto que viene del usuario o de afuera | `TryParse` | `Parse` |
| Convertir texto que tú generaste y es seguro | `Parse` | — |
| Convertir entre tipos variados, o tratar `null` como 0 | `Convert.ToX` | Usarlo creyendo que trunca |
| Mostrar un valor | `ToString()` o interpolación | — |

-----

## Resumen en 5 líneas

1. Implícita: automática, sin pérdida (`int` → `long`, `int` → `double`).
2. Explícita: `(int)3.9` **trunca** a 3; puede desbordar en silencio salvo con `checked`.
3. `Parse` convierte texto y lanza una excepción si falla; `TryParse` devuelve `true`/`false`.
4. `Convert.ToInt32` redondea al par más cercano y trata `null` como 0.
5. Para decimales en texto, cuida la cultura: `CultureInfo.InvariantCulture` para datos de máquina.

-----

## Para profundizar

<details>
<summary>Cast frente a conversión de tipos de referencia</summary>

El cast `(T)x` también se usa con clases, pero ahí no transforma nada: solo cambia **cómo ves** el mismo objeto (por ejemplo, de `Animal` a `Perro`). Si el objeto no es de ese tipo, lanza `InvalidCastException`. Para eso existen `is` y `as`, que se estudian en [Polimorfismo y casting](../04-poo/09-Polimorfismo%20y%20casting.md).

</details>

<details>
<summary>Conversiones definidas por el usuario</summary>

Tus propios tipos pueden definir conversiones con `implicit operator` y `explicit operator`:

```csharp
readonly struct Celsius
{
    public double Grados { get; }
    public Celsius(double g) => Grados = g;

    public static implicit operator double(Celsius c) => c.Grados;
    public static explicit operator Celsius(double d) => new Celsius(d);
}

Celsius t = (Celsius)25.0;   // explícita
double d = t;                // implícita
```

Regla de diseño: implícita solo si nunca falla ni pierde información.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las conversiones implícitas ocurren solas cuando no se pierde información, como de `int` a `double`. Las explícitas requieren un cast, como `(int)3.7`, que trunca a 3. Para convertir texto a número se usa `Parse`, que lanza una excepción si falla, o `TryParse`, que devuelve `false` sin lanzar excepciones, y es lo recomendado para datos del usuario.

### Respuesta ampliada (semi-senior)

Las conversiones numéricas de ampliación son implícitas y las de restricción son explícitas; por defecto se ejecutan en contexto `unchecked`, así que pueden desbordar en silencio, salvo que uses `checked` o la opción del proyecto. El cast entre numéricos trunca hacia cero, mientras que `Convert.ToInt32(double)` y `Math.Round` usan redondeo bancario (`MidpointRounding.ToEven`) por defecto. `Parse` lanza `FormatException`, `OverflowException` o `ArgumentNullException`; `TryParse` evita usar excepciones para el control de flujo, que es costoso. El parseo depende de la cultura actual, por lo que en datos persistidos o de intercambio se usa `CultureInfo.InvariantCulture`.

### Preguntas frecuentes de seguimiento

**1. ¿`(int)3.7` redondea?**
No, trunca a 3. Para redondear se usa `Math.Round`.

**2. ¿Por qué preferir `TryParse` a `Parse` dentro de un `try/catch`?**
Porque lanzar y capturar excepciones es costoso y oscurece la intención. Un formato inválido en una entrada del usuario es un caso esperado, no excepcional.

**3. ¿Qué devuelve `Convert.ToInt32(2.5)`?**
`2`, por el redondeo bancario (al par más cercano).

**4. Si `int x = 256;`, ¿qué da `(byte)x`?**
`0` en contexto `unchecked`; en `checked` lanza `OverflowException`. Ojo: con un literal constante, `(byte)256`, el compilador lo detecta y da `error CS0221`.

-----

## Práctica

**Ejercicio 1.** Sin ejecutar, ¿qué imprime cada línea?

```csharp
Console.WriteLine((int)7.9);
Console.WriteLine((int)Math.Round(7.5));
Console.WriteLine((int)Math.Round(6.5));
Console.WriteLine(Convert.ToInt32(7.9));
Console.WriteLine(int.TryParse("12a", out int n) + " " + n);
```

<details>
<summary>Solución</summary>

```text
7
8
6
8
False 0
```

`Math.Round` usa redondeo bancario: 7.5 → 8 (par) y 6.5 → 6 (par). `TryParse` falla y deja `n` en 0.

</details>

**Ejercicio 2.** Escribe un programa que pida dos números enteros al usuario y muestre su suma. Si alguno no es válido, debe mostrar "Entrada inválida" sin detenerse con un error.

<details>
<summary>Solución</summary>

```csharp
Console.Write("Primer número: ");
bool ok1 = int.TryParse(Console.ReadLine(), out int a);

Console.Write("Segundo número: ");
bool ok2 = int.TryParse(Console.ReadLine(), out int b);

if (ok1 && ok2)
    Console.WriteLine($"Suma: {a + b}");
else
    Console.WriteLine("Entrada inválida");
```

`TryParse` acepta `null` sin lanzar excepciones (devuelve `false`), así que no hace falta el `?? ""`.

</details>

-----

## Siguiente lección

[Números y operadores](04-Numeros%20y%20operadores.md)
