# Valores de retorno y parámetros out

## En una frase

Un método entrega su resultado con `return` (y declara el tipo de ese resultado antes de su nombre); cuando necesitas devolver **más de un valor**, puedes usar parámetros `out` (como `int.TryParse`) o, de forma más moderna y legible, una **tupla**.

-----

## Antes de empezar

Conviene que ya sepas:

* Definir métodos con parámetros, de [Definir y llamar métodos](01-Definir%20y%20llamar%20metodos.md).
* Que los argumentos se pasan como copia, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Valor de retorno:** el resultado que un método entrega a quien lo llamó.
* **Tipo de retorno:** el tipo de ese resultado, escrito antes del nombre del método (`int`, `string`, `void`).
* **`return`:** sentencia que termina el método y, si corresponde, entrega un valor.
* **Parámetro `out`:** parámetro que el método **debe** asignar y que el llamador recibe ya modificado.
* **Parámetro `ref`:** parámetro que el llamador pasa por referencia; el método puede leerlo y modificarlo.
* **Tupla:** grupo de valores, de tipos posiblemente distintos, tratado como una unidad: `(int, string)`.
* **Deconstrucción:** separar una tupla en variables individuales: `var (a, b) = tupla;`.
* **Descarte (`_`):** variable "de usar y tirar" para un valor que no te interesa.

-----

## El problema

¿Qué produce llamar a un método? Depende:

```csharp
Console.WriteLine("¡Hola!");                  // imprime algo, pero no te devuelve nada
double piso = Math.Floor(15.6);               // te DEVUELVE 15
bool ok = int.TryParse("10602", out int n);   // te devuelve true Y además modifica n
```

Hasta ahora tus métodos solo imprimían. Pero un método que calcula un total y lo **imprime** no sirve si necesitas usar ese total en otra cuenta, guardarlo o mostrarlo en otro formato. Necesitas que el método **devuelva** el resultado y que quien lo llama decida qué hacer con él.

-----

## Cómo funciona

### `return` y el tipo de retorno

```csharp
string resultado = Gritar("¿quién está ahí?");
Console.WriteLine(resultado);   // ¿QUIÉN ESTÁ AHÍ?

static string Gritar(string frase)
{
    return frase.ToUpper();
}
```

1. Se llama a `Gritar` con el argumento.
2. `return` calcula `frase.ToUpper()`, **termina el método** y entrega ese valor.
3. En el lugar de la llamada, `Gritar("...")` "se reemplaza" por el valor devuelto, que se guarda en `resultado`.

El **tipo de retorno** (`string`) se declara antes del nombre y es una promesa: el método siempre devolverá un `string`. Si no devuelve nada, el tipo es `void`.

Puedes usar el valor devuelto directamente, sin variable intermedia:

```csharp
Console.WriteLine(Gritar("hola"));                 // anidar llamadas
int menor = Math.Min(3, Math.Max(1, 2));            // el resultado de Max es argumento de Min
```

### `return` termina el método

Todo lo que está después de un `return` ejecutado no se ejecuta. Un método puede tener varios `return`:

```csharp
static string Clasificar(int nota)
{
    if (nota < 0 || nota > 20)
    {
        return "Nota inválida";        // sale aquí si la nota es inválida
    }

    if (nota >= 13)
    {
        return "Aprobado";
    }

    return "Desaprobado";              // el compilador exige un return al final de todos los caminos
}
```

### `return;` en un método `void`

En un método `void`, `return;` (sin valor) sale del método antes de llegar al final:

```csharp
static void Saludar(string? nombre)
{
    if (string.IsNullOrWhiteSpace(nombre))
    {
        Console.WriteLine("No hay a quién saludar");
        return;                         // termina el método, no el programa
    }

    Console.WriteLine($"Hola, {nombre}");
}
```

`return;` termina **el método**, no el programa. Solo cuando está en el punto de entrada (`Main` o las top-level statements) el programa termina, porque ese método era el programa.

### Parámetros `out`: devolver un valor extra

Un método solo tiene **un** valor de retorno. `out` es una forma de devolver datos adicionales a través de los parámetros. El ejemplo clásico es `TryParse`:

```csharp
public static bool TryParse(string? s, out int result);   // firma (simplificada)
```

```csharp
bool exito = int.TryParse("10602", out int numero);   // exito = true, numero = 10602
bool exito2 = int.TryParse("!!!", out int numero2);   // exito2 = false, numero2 = 0
```

* El valor de retorno (`bool`) dice **si funcionó**.
* El parámetro `out` entrega **el resultado**.

Escribe tus propios métodos con `out`:

```csharp
string texto = Gritar("garrrr", out bool fueGritado);
Console.WriteLine($"{texto} {fueGritado}");   // GARRRR True

static string Gritar(string frase, out bool fueGritado)
{
    fueGritado = true;                // obligatorio: asignar el out antes de salir
    return frase.ToUpper();
}
```

Reglas de `out`:

* Se escribe `out` **en la definición y en la llamada**.
* El método **debe asignar** el parámetro antes de terminar, en todos los caminos.
* El método **no puede leerlo** antes de asignarlo.
* En la llamada puedes declarar la variable ahí mismo (`out int numero`) o pasar una existente (`out numero`).
* Si no te interesa el valor, usa un descarte: `int.TryParse(texto, out _)`.

### `ref`: modificar la variable del llamador

Con `ref`, el método recibe **la variable misma**, no una copia. Lo que haga se ve afuera:

```csharp
int puntos = 10;
Duplicar(ref puntos);
Console.WriteLine(puntos);    // 20

static void Duplicar(ref int valor)
{
    valor *= 2;
}
```

| Modificador | El llamador debe inicializarla | El método debe asignarla | El método puede modificarla |
| --- | --- | --- | --- |
| (ninguno) | Sí | No | Solo su copia |
| `ref` | Sí | No | Sí, y se ve afuera |
| `out` | No | **Sí** | Sí, y se ve afuera |
| `in` | Sí | No | **No** (solo lectura) |

`ref` y `out` son herramientas puntuales. En código de aplicación, preferirás casi siempre devolver un valor.

### Tuplas: devolver varios valores de forma legible

Una tupla agrupa varios valores sin tener que crear una clase:

```csharp
var persona = (Id: 1, Nombre: "Carlos", EsMayorDeEdad: true);

Console.WriteLine(persona.Nombre);   // Carlos (acceso por nombre)
Console.WriteLine(persona.Item2);    // Carlos (acceso por posición: Item1, Item2...)
```

Como tipo de retorno, son la forma moderna de devolver varios valores:

```csharp
var (minimo, maximo) = ObtenerExtremos(new[] { 4, 9, 1, 7 });
Console.WriteLine($"Mínimo: {minimo}, máximo: {maximo}");   // Mínimo: 1, máximo: 9

static (int Min, int Max) ObtenerExtremos(int[] numeros)
{
    int min = numeros[0], max = numeros[0];
    foreach (int n in numeros)
    {
        if (n < min) min = n;
        if (n > max) max = n;
    }
    return (min, max);
}
```

* `(int Min, int Max)` es el tipo de retorno: una tupla con dos `int` con nombre.
* `return (min, max);` construye la tupla.
* `var (minimo, maximo) = ...` **deconstruye** la tupla en dos variables.
* Puedes ignorar una parte con un descarte: `var (_, maximo) = ObtenerExtremos(...)`.

Dos detalles importantes que suelen enseñarse mal:

* Las tuplas **no son inmutables**: sus elementos se pueden modificar (`persona.Nombre = "Ana";` compila). Son tipos de valor (`ValueTuple`), así que se copian al asignarlas.
* Los nombres (`Nombre`, `Min`) existen **solo para el compilador**: en ejecución, los elementos se llaman `Item1`, `Item2`... Por eso no puedes obtenerlos con reflexión.

### `out` frente a tupla

```csharp
// Con out
bool ok = TryDividir(10, 3, out int cociente, out int resto);

// Con tupla
var (cociente2, resto2) = Dividir(10, 3);

static bool TryDividir(int a, int b, out int cociente, out int resto)
{
    if (b == 0) { cociente = 0; resto = 0; return false; }
    cociente = a / b;
    resto = a % b;
    return true;
}

static (int Cociente, int Resto) Dividir(int a, int b) => (a / b, a % b);
```

La convención en .NET es: patrón **`TryX(..., out resultado)`** cuando la operación puede fallar de forma esperada; **tupla** cuando simplemente quieres devolver varios datos relacionados.

-----

## Ejemplo completo

```csharp
string[] entradas = { "15", "abc", "8", "-3", "20" };

var (validos, invalidos) = Clasificar(entradas);
Console.WriteLine($"Válidos: {validos.Length}, inválidos: {invalidos}");

var (promedio, mayor) = Estadisticas(validos);
Console.WriteLine($"Promedio: {promedio:F2}, mayor: {mayor}");

if (TryBuscar(validos, 8, out int posicion))
{
    Console.WriteLine($"El 8 está en la posición {posicion}");
}

static (int[] Validos, int Invalidos) Clasificar(string[] textos)
{
    var numeros = new List<int>();
    int invalidos = 0;

    foreach (string t in textos)
    {
        if (int.TryParse(t, out int n) && n >= 0)
            numeros.Add(n);
        else
            invalidos++;
    }

    return (numeros.ToArray(), invalidos);
}

static (double Promedio, int Mayor) Estadisticas(int[] numeros)
{
    if (numeros.Length == 0) return (0, 0);

    int suma = 0, mayor = numeros[0];
    foreach (int n in numeros)
    {
        suma += n;
        if (n > mayor) mayor = n;
    }
    return ((double)suma / numeros.Length, mayor);
}

static bool TryBuscar(int[] datos, int buscado, out int posicion)
{
    for (int i = 0; i < datos.Length; i++)
    {
        if (datos[i] == buscado)
        {
            posicion = i;
            return true;
        }
    }
    posicion = -1;
    return false;
}
```

Salida:

```text
Válidos: 3, inválidos: 2
Promedio: 14.33, mayor: 20
El 8 está en la posición 1
```

`List<int>` es una colección que crece; se estudia más adelante. Aquí solo se usa para ir agregando los números válidos.

-----

## Errores comunes

**1. No todos los caminos devuelven un valor.**
Qué pasa: `error CS0161: 'Clasificar(int)': not all code paths return a value`.
Por qué: hay algún camino (por ejemplo, un `if` sin `else`) que llega al final sin `return`.
Arreglo: agrega un `return` final que cubra el caso restante.

**2. Devolver un tipo distinto al declarado.**
Qué pasa: `error CS0029: Cannot implicitly convert type 'string' to 'int'`.
Por qué: el tipo de retorno es una promesa que el compilador verifica.
Arreglo: devuelve el tipo correcto o cambia el tipo de retorno.

**3. Devolver un valor desde un método `void`.**
Qué pasa: `error CS0127: Since 'Saludar(string)' returns void, a return keyword must not be followed by an object expression`.
Por qué: `void` significa que no se devuelve nada.
Arreglo: cambia `void` por el tipo adecuado, o usa `return;` sin valor.

**4. `return;` sin valor en un método que debe devolver algo.**
Qué pasa: `error CS0126: An object of a type convertible to 'int' is required`.
Por qué: falta el valor.
Arreglo: `return valor;`.

**5. Olvidar el tipo de retorno (dentro de una clase).**
Qué pasa: `error CS1520: Method must have a return type`.
Por qué: todo método declara su tipo de retorno o `void`.
Arreglo: `static void Metodo()` o `static int Metodo()`.

**6. No asignar un parámetro `out`.**
Qué pasa: `error CS0177: The out parameter 'posicion' must be assigned to before control leaves the current method`.
Por qué: el llamador espera recibir un valor siempre.
Arreglo: asígnalo en todos los caminos, incluso en los de error.

**7. Olvidar `out` en la llamada.**
Qué pasa: `error CS1620: Argument 2 must be passed with the 'out' keyword`.
Por qué: el modificador debe aparecer también al llamar, para que sea visible que la variable se va a modificar.
Arreglo: `Metodo(x, out resultado)`.

**8. Ignorar el valor devuelto.**
Qué pasa: compila, pero el resultado se pierde (`texto.Trim();`, `Calcular(5);`).
Por qué: llamar a un método no guarda su resultado en ningún lado.
Arreglo: asígnalo: `texto = texto.Trim();`.

-----

## Según la versión de C#

* **C# 7:** `out var` en la llamada (`out int n`), descartes `_`, tuplas con nombre (`ValueTuple`) y deconstrucción.
* **C# 7.2:** parámetros `in` (solo lectura por referencia).
* **C# 7.3:** comparación de tuplas con `==` y `!=`.
* Antes de C# 7 existía `System.Tuple<T1, T2>` (una clase inmutable con `Item1`, `Item2`), mucho menos práctica. Si la ves en código antiguo, es otra cosa distinta de `(int, string)`.

-----

## Cuándo sí y cuándo no

| Necesitas... | Usa |
| --- | --- |
| Devolver un resultado | `return` con el tipo adecuado |
| Indicar éxito o fracaso + resultado en una operación que puede fallar | Patrón `TryX(..., out T resultado)` |
| Devolver 2 o 3 valores relacionados, uso local | Tupla con nombres |
| Devolver muchos datos o datos que viajan por varias capas | Una clase o un `record` |
| Que un método modifique la variable del llamador | `ref`, con moderación |

**Evita:**

* Métodos con varios `out`: son difíciles de leer. Una tupla o un tipo propio son más claros.
* Tuplas en APIs públicas o que cruzan varias capas: los nombres de una clase o un record documentan mejor.

-----

## Resumen en 5 líneas

1. El tipo de retorno va antes del nombre; `return valor;` termina el método y entrega el valor.
2. `void` = sin valor de retorno; `return;` sale antes de un método `void`.
3. Todos los caminos de un método no `void` deben terminar en `return` (CS0161).
4. `out`: el método debe asignarlo y el llamador escribe `out` en la llamada; es la base del patrón `TryParse`.
5. Las tuplas `(int Min, int Max)` devuelven varios valores; se deconstruyen con `var (a, b) = ...`.

-----

## Para profundizar

<details>
<summary>Recorrer los elementos de una tupla</summary>

Como los nombres de una tupla solo existen al compilar, recorrerla con reflexión (`GetProperties()`) **no funciona**: `ValueTuple` tiene campos, no propiedades, y se llaman `Item1`, `Item2`... Si de verdad necesitas recorrerla, usa la interfaz `ITuple`:

```csharp
using System.Runtime.CompilerServices;

var tupla = (Nombre: "Carlos", Edad: 30, Ciudad: "Lima");
ITuple t = tupla;

for (int i = 0; i < t.Length; i++)
{
    Console.WriteLine(t[i]);    // Carlos, 30, Lima (sin los nombres)
}
```

Si necesitas recorrer los datos por nombre, probablemente no deberías usar una tupla sino un diccionario o una clase.

</details>

<details>
<summary>ref returns y ref locals</summary>

Desde C# 7, un método puede devolver una **referencia** a una variable en lugar de una copia:

```csharp
ref int BuscarRef(int[] datos, int valor)
{
    for (int i = 0; i < datos.Length; i++)
        if (datos[i] == valor) return ref datos[i];
    throw new InvalidOperationException("No encontrado");
}

int[] numeros = { 1, 2, 3 };
ref int celda = ref BuscarRef(numeros, 2);
celda = 99;   // numeros ahora es { 1, 99, 3 }
```

Se usa en código de alto rendimiento para evitar copias de structs grandes. En aplicaciones normales no lo necesitarás.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`return` termina el método y devuelve un valor del tipo declarado; los métodos `void` no devuelven nada. Para devolver más de un valor existen los parámetros `out`, que el método debe asignar, como en `int.TryParse`, o las tuplas, como `(int Min, int Max)`, que se pueden deconstruir en variables.

### Respuesta ampliada (semi-senior)

`out` y `ref` pasan argumentos por referencia: `ref` exige que la variable esté inicializada y permite leerla y escribirla; `out` no exige inicialización, pero obliga al método a asignarla; `in` es por referencia y de solo lectura, útil para structs grandes. El patrón `TryX` con `out` es la convención de .NET para operaciones que pueden fallar sin excepciones. Las tuplas de C# 7 son `ValueTuple`: tipos de valor mutables con nombres de elementos que solo existen en tiempo de compilación (se preservan como metadatos en las firmas públicas, pero no en el objeto). Para modelos de datos que cruzan capas, se prefiere un `record`.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `ref` y `out`?**
`ref` requiere que la variable esté inicializada antes de la llamada; `out` no, pero el método está obligado a asignarla.

**2. ¿Las tuplas son inmutables?**
`ValueTuple` (la sintaxis `(a, b)`) no: sus elementos son campos públicos modificables. La antigua clase `System.Tuple` sí era inmutable.

**3. ¿Cuándo usarías una tupla y cuándo un record?**
Tupla para devolver valores relacionados en un ámbito local o privado; record cuando el dato tiene identidad conceptual, viaja entre capas o forma parte de una API pública.

-----

## Práctica

**Ejercicio 1.** Escribe un método `CalcularIMC(double pesoKg, double alturaM)` que devuelva el índice de masa corporal (`peso / altura²`) y otro método `Categoria(double imc)` que devuelva `"Bajo peso"` (< 18.5), `"Normal"` (< 25), `"Sobrepeso"` (< 30) u `"Obesidad"`. Combínalos para imprimir el resultado de 70 kg y 1.75 m.

<details>
<summary>Solución</summary>

```csharp
double imc = CalcularIMC(70, 1.75);
Console.WriteLine($"IMC: {imc:F1} → {Categoria(imc)}");   // IMC: 22.9 → Normal

static double CalcularIMC(double pesoKg, double alturaM) => pesoKg / (alturaM * alturaM);

static string Categoria(double imc)
{
    if (imc < 18.5) return "Bajo peso";
    if (imc < 25) return "Normal";
    if (imc < 30) return "Sobrepeso";
    return "Obesidad";
}
```

</details>

**Ejercicio 2.** Escribe `TryParseFecha(string texto, out int dia, out int mes)` que acepte textos con el formato `"dd/mm"`. Debe devolver `false` (y asignar 0 a los dos `out`) si el formato no es válido o si el día o el mes están fuera de rango. Después, reescríbelo devolviendo una tupla `(bool Ok, int Dia, int Mes)`.

<details>
<summary>Solución</summary>

```csharp
if (TryParseFecha("25/12", out int d, out int m))
    Console.WriteLine($"Día {d}, mes {m}");

var (ok, dia, mes) = ParseFecha("31/02");
Console.WriteLine(ok ? $"Día {dia}, mes {mes}" : "Fecha inválida");

static bool TryParseFecha(string texto, out int dia, out int mes)
{
    dia = 0;
    mes = 0;

    string[] partes = texto.Split('/');
    if (partes.Length != 2) return false;
    if (!int.TryParse(partes[0], out int d) || !int.TryParse(partes[1], out int m)) return false;
    if (m is < 1 or > 12 || d < 1 || d > DateTime.DaysInMonth(2024, m)) return false;

    dia = d;
    mes = m;
    return true;
}

static (bool Ok, int Dia, int Mes) ParseFecha(string texto)
{
    bool ok = TryParseFecha(texto, out int dia, out int mes);
    return (ok, dia, mes);
}
```

Asignar los `out` al principio garantiza que todos los caminos los asignan. Se usa 2024 (bisiesto) para admitir el 29/02; el 31/02 es inválido en cualquier año.

</details>

-----

## Siguiente lección

[Expresiones lambda](04-Expresiones%20lambda.md)
