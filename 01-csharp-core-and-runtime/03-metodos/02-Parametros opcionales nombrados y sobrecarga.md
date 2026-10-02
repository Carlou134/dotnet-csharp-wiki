# Parámetros opcionales, nombrados y sobrecarga

## En una frase

Un **parámetro opcional** tiene un valor por defecto y se puede omitir al llamar; un **argumento con nombre** indica a qué parámetro va (`d: 4`) sin depender del orden; y la **sobrecarga** permite tener varios métodos con el mismo nombre pero distintos parámetros.

-----

## Antes de empezar

Conviene que ya sepas:

* Definir métodos con parámetros y llamarlos con argumentos, de [Definir y llamar métodos](01-Definir%20y%20llamar%20metodos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Parámetro opcional:** parámetro con un valor por defecto; el argumento puede omitirse.
* **Valor por defecto:** el valor que toma un parámetro opcional si no se pasa.
* **Argumento posicional:** argumento que se asigna por su posición en la llamada.
* **Argumento con nombre:** argumento que indica el nombre del parámetro: `Metodo(edad: 30)`.
* **Sobrecarga (*overload*):** cada una de las versiones de un método con el mismo nombre y distintos parámetros.
* **Firma:** nombre del método + tipos (y orden) de sus parámetros. Es lo que distingue una sobrecarga de otra.
* **`params`:** modificador que permite pasar una cantidad variable de argumentos.

-----

## El problema

Tienes un método para imprimir mensajes:

```csharp
static void Mostrar(string mensaje, string puntuacion, bool mayusculas, int repeticiones)
```

El 90% de las veces quieres `"."`, sin mayúsculas y una sola repetición. Obligar a escribir siempre los cuatro argumentos es molesto y, con muchos parámetros del mismo tipo, es fácil confundir el orden: ¿`Mostrar("Hola", "!", true, 2)` o `Mostrar("Hola", "!", 2, true)`?

Y otro caso: quieres sumar dos enteros, tres enteros o dos decimales. ¿Tres nombres distintos (`SumarDos`, `SumarTres`, `SumarDecimales`)? .NET lo resuelve con un solo nombre: `Math.Round` tiene 8 versiones distintas.

-----

## Cómo funciona

### Parámetros opcionales

Se asigna un valor por defecto con `=` en la definición:

```csharp
Mostrar("Tengo hambre", "!");   // Tengo hambre!
Mostrar("Tengo hambre");        // Tengo hambre.

static void Mostrar(string mensaje, string puntuacion = ".")
{
    Console.WriteLine(mensaje + puntuacion);
}
```

Reglas:

* Los parámetros opcionales van **al final**, después de todos los obligatorios.
* El valor por defecto debe ser una **constante de compilación**: un literal, una `const`, `null` o `default`. `DateTime.Now` no sirve.

```csharp
static void A(int x = 0, int y) { }            // error CS1737: los opcionales deben ir después de los obligatorios
static void B(DateTime fecha = DateTime.Now) { } // error CS1736: el valor por defecto debe ser una constante
static void C(DateTime? fecha = null) { }        // ✅ y dentro: var f = fecha ?? DateTime.Now;
```

### Argumentos con nombre

Tienes un método con cinco opcionales y solo quieres cambiar `d`:

```csharp
static void Configurar(int a = 0, int b = 0, int c = 0, int d = 0, int e = 0)
{
    Console.WriteLine($"a={a} b={b} c={c} d={d} e={e}");
}
```

```csharp
Configurar(4);                   // a=4 ... ← asignó a 'a', no a 'd'
Configurar(d: 4);                // a=0 b=0 c=0 d=4 e=0
Configurar(d: 4, b: 1, a: 2);    // con nombre, el orden no importa
```

Puedes **mezclar** posicionales y con nombre. Los posicionales van primero:

```csharp
Configurar(2, 1, d: 4);          // ✅ a=2, b=1, d=4
Configurar(d: 4, 2, 1);          // ❌ error CS8323: el argumento con nombre 'd' está fuera de posición
```

Los argumentos con nombre también hacen **más legible** cualquier llamada con literales:

```csharp
EnviarCorreo("ana@mail.com", true, false);                          // ¿qué significan true y false?
EnviarCorreo("ana@mail.com", urgente: true, conCopia: false);       // se explica sola
```

### Sobrecarga de métodos

Varios métodos pueden compartir **el mismo nombre** si sus parámetros son distintos en **cantidad**, **tipo** u **orden de tipos**:

```csharp
Console.WriteLine(Calculadora.Sumar(2, 3));         // 5     → Sumar(int, int)
Console.WriteLine(Calculadora.Sumar(2, 3, 4));      // 9     → Sumar(int, int, int)
Console.WriteLine(Calculadora.Sumar(2.5, 3.5));     // 6     → Sumar(double, double)

static class Calculadora
{
    public static int Sumar(int a, int b) => a + b;
    public static int Sumar(int a, int b, int c) => a + b + c;
    public static double Sumar(double a, double b) => a + b;
}
```

* La sintaxis `=> a + b` es una forma corta de escribir un método de una línea; se explica en [Expresiones lambda](04-Expresiones%20lambda.md).
* Las sobrecargas van **dentro de una clase** (aquí, `Calculadora`). `public` permite usarlas desde fuera de la clase y `static` permite llamarlas sin crear un objeto: `Calculadora.Sumar(...)`. Las dos palabras se estudian en [POO](../04-poo/README.md).

**¿Por qué una clase?** Con top-level statements, los métodos escritos sueltos en `Program.cs` son **funciones locales**, y las funciones locales **no admiten sobrecarga**:

```csharp
static int Sumar(int a, int b) => a + b;
static int Sumar(int a, int b, int c) => a + b + c;   // error CS0128: ya hay una variable o función local llamada 'Sumar'
```

El compilador elige la sobrecarga **al compilar**, mirando los argumentos. Esa elección se llama *resolución de sobrecarga*.

Ya usaste sobrecargas sin saberlo:

```csharp
Math.Round(3.14159);        // 3     → Round(double)
Math.Round(3.14159, 2);     // 3.14  → Round(double, int)
Console.WriteLine(42);      // WriteLine(int)
Console.WriteLine("hola");  // WriteLine(string)
```

### Qué NO cuenta como sobrecarga distinta

La **firma** es el nombre + los tipos de los parámetros. **No** incluye:

* El **tipo de retorno**.
* Los **nombres** de los parámetros.

```csharp
static class Ejemplos
{
    public static int Obtener(int id) => id;
    public static string Obtener(int id) => id.ToString();  // error CS0111: ya existe un miembro 'Obtener' con los mismos tipos de parámetros

    public static void Mostrar(int edad) { }
    public static void Mostrar(int cantidad) { }            // error CS0111: mismo tipo, distinto nombre no alcanza
}
```

### Ambigüedad

Si dos sobrecargas sirven igual de bien, el compilador no elige:

```csharp
Ejemplos.Procesar(1, 2);   // error CS0121: la llamada es ambigua entre Procesar(int, double) y Procesar(double, int)

static class Ejemplos
{
    public static void Procesar(int a, double b) { }
    public static void Procesar(double a, int b) { }
}
```

Las sobrecargas combinadas con parámetros opcionales también pueden volverse confusas. Úsalas con criterio.

### `params`: cantidad variable de argumentos

Con `params`, el último parámetro acepta **cero o más** argumentos sueltos:

```csharp
Console.WriteLine(SumarTodos());               // 0
Console.WriteLine(SumarTodos(1, 2));           // 3
Console.WriteLine(SumarTodos(1, 2, 3, 4, 5));  // 15

static int SumarTodos(params int[] numeros)
{
    int total = 0;
    foreach (int n in numeros) total += n;
    return total;
}
```

* Dentro del método, `numeros` es un array normal.
* Solo puede haber un `params` y debe ser el **último** parámetro.
* `Console.WriteLine("{0} {1} {2}", a, b, c)` y `string.Join(", ", a, b, c)` funcionan así.

-----

## Ejemplo completo

```csharp
// Sobrecargas para formatear precios en distintos tipos
Console.WriteLine(Formato.Precio(1500));                         // S/ 1,500.00
Console.WriteLine(Formato.Precio(1500.5m));                      // S/ 1,500.50
Console.WriteLine(Formato.Precio(1500.5m, simbolo: "$"));        // $ 1,500.50
Console.WriteLine(Formato.Precio(1500.5m, decimales: 0));        // S/ 1,501

// params + argumentos con nombre (una función local: no necesita sobrecarga)
RegistrarLog("Pedido creado", nivel: "INFO");
RegistrarLog("Stock bajo", "WARN", "producto=42", "stock=3");

static void RegistrarLog(string mensaje, string nivel = "DEBUG", params string[] datos)
{
    string extra = datos.Length > 0 ? $" [{string.Join(", ", datos)}]" : "";
    Console.WriteLine($"[{nivel}] {mensaje}{extra}");
}

static class Formato
{
    public static string Precio(int monto) => Precio((decimal)monto);

    public static string Precio(decimal monto, string simbolo = "S/", int decimales = 2)
    {
        return $"{simbolo} {monto.ToString($"N{decimales}")}";
    }
}
```

Salida (con cultura en inglés para los separadores):

```text
S/ 1,500.00
S/ 1,500.50
$ 1,500.50
S/ 1,501
[INFO] Pedido creado
[WARN] Stock bajo [producto=42, stock=3]
```

La sobrecarga de `int` **reutiliza** la de `decimal` en lugar de duplicar la lógica. Y `decimales: 0` usa un argumento con nombre para saltar `simbolo`.

-----

## Errores comunes

**1. Parámetro opcional antes de uno obligatorio.**
Qué pasa: `error CS1737: Optional parameters must appear after all required parameters`.
Por qué: el compilador no sabría a qué parámetro corresponde cada argumento.
Arreglo: mueve los opcionales al final.

**2. Valor por defecto no constante.**
Qué pasa: `error CS1736: Default parameter value for 'fecha' must be a compile-time constant`.
Por qué: el valor por defecto se incrusta en el código del que llama al compilar.
Arreglo: usa `null` o `default` como valor por defecto y calcula el valor real dentro del método.

**3. Sobrecargas que solo difieren en el tipo de retorno.**
Qué pasa: `error CS0111: Type 'Program' already defines a member called 'Obtener' with the same parameter types`.
Por qué: el tipo de retorno no forma parte de la firma.
Arreglo: cambia los parámetros o usa nombres distintos (`ObtenerTexto`, `ObtenerNumero`).

**4. Llamada ambigua.**
Qué pasa: `error CS0121: The call is ambiguous between the following methods or properties`.
Por qué: más de una sobrecarga encaja igual de bien.
Arreglo: haz explícito el tipo (`Procesar(1, 2.0)`) o rediseña las sobrecargas.

**5. Argumento con nombre fuera de posición antes de uno posicional.**
Qué pasa: `error CS8323: Named argument 'd' is used out-of-position but is followed by an unnamed argument`.
Por qué: tras un argumento con nombre fuera de su posición, el compilador ya no puede asignar los posicionales.
Arreglo: pon los posicionales primero.

**6. Escribir mal el nombre de un argumento.**
Qué pasa: `error CS1739: The best overload for 'Configurar' does not have a parameter named 'D'`.
Por qué: los nombres distinguen mayúsculas de minúsculas.
Arreglo: usa el nombre exacto del parámetro.

**7. Sobrecargar métodos escritos sueltos en `Program.cs`.**
Qué pasa: `error CS0128: A local variable or function named 'Sumar' is already defined in this scope`.
Por qué: con top-level statements, esos métodos son funciones locales, y las funciones locales no se pueden sobrecargar.
Arreglo: mueve las sobrecargas a una clase (por ejemplo, `static class Calculadora`).

-----

## Según la versión de C#

* **C# 4:** parámetros opcionales y argumentos con nombre (antes solo existía la sobrecarga para simularlos).
* **C# 7.2:** argumentos con nombre seguidos de posicionales, siempre que estén en su posición correcta.
* **C# 12:** parámetros por defecto en lambdas (`(int x = 1) => x * 2`).
* **C# 13:** `params` con cualquier colección, no solo arrays (`params List<int>`, `params ReadOnlySpan<int>`), lo que evita crear un array en cada llamada.

-----

## Cuándo sí y cuándo no

**Usa parámetros opcionales cuando:**

* Hay un valor por defecto razonable y la mayoría de las llamadas lo usa.

**Usa sobrecarga cuando:**

* El método trabaja con **tipos distintos** (`int`, `decimal`, `string`).
* Las variantes tienen lógica realmente diferente.

**Usa argumentos con nombre cuando:**

* Pasas literales cuyo significado no es obvio (`true`, `0`, `null`).
* Quieres saltar parámetros opcionales.

**Evita:**

* Más de 3 o 4 parámetros opcionales: agrúpalos en un objeto de opciones.
* Parámetros opcionales en métodos **públicos de librerías** que cambian seguido (ver "Para profundizar").

-----

## Resumen en 5 líneas

1. `Metodo(string x, int y = 0)`: los opcionales tienen un valor constante y van al final.
2. `Metodo(y: 5, x: "a")`: con nombre, el orden no importa; los posicionales van antes.
3. Sobrecarga: mismo nombre, distintos tipos o cantidad de parámetros.
4. El tipo de retorno y los nombres de los parámetros no distinguen sobrecargas.
5. `params int[] numeros` acepta cero o más argumentos sueltos.

-----

## Para profundizar

<details>
<summary>Los valores por defecto se copian en el llamador</summary>

Cuando compilas `Mostrar("hola")`, el compilador escribe en **tu** código `Mostrar("hola", ".")`. Si una librería cambia el valor por defecto a `"!"` y no recompilas tu proyecto, seguirás usando `"."`. Por eso, en APIs públicas de librerías, muchas veces se prefieren las sobrecargas a los opcionales: con sobrecargas, el valor vive dentro de la librería.

</details>

<details>
<summary>Cómo elige el compilador una sobrecarga</summary>

El compilador busca la sobrecarga "mejor" según reglas: prefiere coincidencias exactas de tipo antes que conversiones implícitas, y conversiones más específicas (`int` → `long`) antes que más generales (`int` → `object`). Por ejemplo, con `M(long)` y `M(object)`, la llamada `M(5)` elige `M(long)`. Si no hay una única "mejor", da el error CS0121. Desde C# 13, los autores de librerías pueden desempatar con el atributo `OverloadResolutionPriority`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los parámetros opcionales tienen un valor por defecto y van al final, así que se pueden omitir al llamar. Los argumentos con nombre permiten indicar a qué parámetro va cada valor sin respetar el orden. La sobrecarga es tener varios métodos con el mismo nombre y distintos parámetros, y el compilador elige cuál usar según los argumentos.

### Respuesta ampliada (semi-senior)

La resolución de sobrecarga es estática, en tiempo de compilación, y considera la cantidad, los tipos y los modificadores (`ref`/`out`/`in`) de los parámetros, no el tipo de retorno; si no hay una mejor candidata única, da CS0121. Los valores por defecto deben ser constantes de compilación y se incrustan en el sitio de llamada, lo que tiene implicaciones de versionado en librerías públicas. `params` crea un array por llamada (salvo en C# 13 con `params ReadOnlySpan<T>`). La sobrecarga es polimorfismo estático (*ad hoc*), distinto del polimorfismo de subtipos con `virtual`/`override`.

### Preguntas frecuentes de seguimiento

**1. ¿Se puede sobrecargar solo cambiando el tipo de retorno?**
No. El tipo de retorno no forma parte de la firma.

**2. ¿Sobrecarga es lo mismo que sobrescritura (*override*)?**
No. Sobrecarga: mismo nombre, distintos parámetros, se resuelve al compilar. Sobrescritura: una clase hija redefine un método `virtual` de la clase base con la misma firma, y se resuelve al ejecutar. Se ve en [Virtual, override y clases abstractas](../04-poo/07-Virtual%20override%20y%20clases%20abstractas.md).

**3. ¿Opcionales o sobrecargas?**
Opcionales para variantes simples dentro de tu propio código; sobrecargas cuando cambian los tipos, la lógica es distinta o se trata de una API pública versionada.

-----

## Práctica

**Ejercicio 1.** Escribe un método `Saludar` con un parámetro obligatorio `nombre`, y dos opcionales: `saludo` (por defecto `"Hola"`) y `exclamacion` (por defecto `false`). Llámalo para obtener estas tres salidas:

```text
Hola, Ana.
Buenos días, Luis.
Hola, Eva!
```

<details>
<summary>Solución</summary>

```csharp
Saludar("Ana");
Saludar("Luis", "Buenos días");
Saludar("Eva", exclamacion: true);

static void Saludar(string nombre, string saludo = "Hola", bool exclamacion = false)
{
    string fin = exclamacion ? "!" : ".";
    Console.WriteLine($"{saludo}, {nombre}{fin}");
}
```

La tercera llamada salta `saludo` gracias al argumento con nombre.

</details>

**Ejercicio 2.** ¿Cuáles de estos pares de métodos pueden coexistir en la misma clase?

```csharp
// A
void M(int x) { }
void M(long x) { }

// B
int M(string s) { return 0; }
void M(string texto) { }

// C
void M(int a, string b) { }
void M(string a, int b) { }
```

<details>
<summary>Solución</summary>

* **A:** sí. Los tipos de los parámetros son distintos.
* **B:** no. Solo cambian el tipo de retorno y el nombre del parámetro (CS0111).
* **C:** sí. El orden de los tipos es distinto.

</details>

-----

## Siguiente lección

[Valores de retorno y parámetros out](03-Valores%20de%20retorno%20y%20parametros%20out.md)
