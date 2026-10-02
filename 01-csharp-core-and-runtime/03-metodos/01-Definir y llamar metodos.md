# Definir y llamar métodos

## En una frase

Un método es un bloque de código con **nombre** que realiza una tarea; se **define** una vez, con parámetros que reciben datos, y se **llama** las veces que haga falta pasándole argumentos.

-----

## Antes de empezar

Conviene que ya sepas:

* Variables, condicionales y bucles: [Control de flujo](../02-control-de-flujo/README.md).
* Que ya usaste métodos como `Console.WriteLine()`, `Math.Max()` o `texto.ToUpper()`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Método:** bloque de código con nombre que realiza una tarea. En C#, toda función pertenece a un tipo, así que se llama método.
* **Definir (declarar) un método:** escribir su firma y su cuerpo.
* **Llamar (invocar) un método:** ejecutarlo escribiendo su nombre seguido de paréntesis.
* **Parámetro:** variable declarada en la definición del método que recibe un valor.
* **Argumento:** el valor concreto que pasas al llamar el método.
* **Cuerpo:** el código entre llaves que se ejecuta al llamar al método.
* **Firma:** el nombre del método más los tipos de sus parámetros.
* **`void`:** indica que el método no devuelve ningún valor.
* **Ámbito (*scope*):** la zona del código donde un nombre existe y se puede usar.

-----

## El problema

Imagina preparar una hamburguesa:

1. Coloca el pan abajo.
2. Agrega la carne.
3. Agrega los pepinillos.
4. Coloca el pan arriba.

Si cada vez que alguien pide una hamburguesa tuvieras que dictar los cuatro pasos, sería tedioso, lento y propenso a errores. Lo que haces en la vida real es ponerle un **nombre** al procedimiento ("hamburguesa clásica") y pedirlo por ese nombre.

En el código pasa lo mismo: si copias el mismo bloque en cinco lugares y después encuentras un error, tienes que corregirlo cinco veces (y seguro te olvidas de uno). Los métodos permiten escribir la lógica **una vez** y reutilizarla.

-----

## Cómo funciona

### Llamar a un método

Ya lo hiciste muchas veces. Se escribe el nombre y, entre paréntesis, los argumentos:

```csharp
Console.WriteLine("¡Tengo hambre!");   // un argumento: el texto a imprimir
Math.Min(3, 5);                         // dos argumentos: los números a comparar
"beatriz".Substring(0, 3);              // método de un string: devuelve "bea"
Console.WriteLine();                    // sin argumentos: los paréntesis van igual
```

Los paréntesis son los que **ejecutan** el método. Sin ellos, solo estás nombrándolo.

### Definir un método

En un programa con top-level statements, defines métodos así:

```csharp
SaludarConEntusiasmo();   // llamada
SaludarConEntusiasmo();   // se puede llamar las veces que quieras

static void SaludarConEntusiasmo()
{
    Console.WriteLine("¡Hola!");
    Console.WriteLine("¡Qué bueno verte!");
}
```

Partes de la definición:

```text
static void SaludarConEntusiasmo ( )
  │     │          │             │
  │     │          │             └─ lista de parámetros (vacía)
  │     │          └─ nombre, en PascalCase y normalmente un verbo
  │     └─ tipo de retorno: void = no devuelve nada
  └─ modificador: no depende de un objeto (se explica en Miembros estáticos)
{
    cuerpo
}
```

* El nombre usa **PascalCase** (`CalcularTotal`, `EnviarCorreo`) y suele ser un **verbo**: un método *hace* algo.
* El cuerpo va entre llaves y se ejecuta cada vez que llamas al método.
* Un método se puede llamar **antes o después** de su definición en el archivo: el compilador lo encuentra igual.

### Dónde viven los métodos

En C#, un método siempre pertenece a un tipo (una clase, un struct...). Hay dos situaciones habituales:

**1. Con top-level statements** (como en el ejemplo anterior), los métodos que escribes en `Program.cs` son en realidad **funciones locales** del método de entrada generado. Pueden llevar `static` o no.

**2. Dentro de una clase**, con un `Main` explícito:

```csharp
class Program
{
    static void Main()
    {
        SaludarConEntusiasmo();
    }

    static void SaludarConEntusiasmo()
    {
        Console.WriteLine("¡Hola!");
    }
}
```

Aquí `static` **sí importa**: desde un método `static` como `Main` solo puedes llamar directamente a otros métodos `static`. Si quitas `static` de `SaludarConEntusiasmo`, obtienes `error CS0120`. La razón se explica en [Miembros estáticos](../04-poo/05-Miembros%20estaticos.md); por ahora, en programas de consola, márcalos como `static`.

### Parámetros: darle datos al método

Un método sin datos siempre hace lo mismo. Con **parámetros**, se adapta a lo que le pases:

```csharp
Saludar("Ana");
Saludar("Luis");

static void Saludar(string nombre)
{
    Console.WriteLine($"Hola, {nombre}");
}
```

* `nombre` es un **parámetro**: una variable que existe solo dentro del método.
* `"Ana"` y `"Luis"` son **argumentos**: los valores concretos de cada llamada.

Varios parámetros se separan con comas, cada uno con su tipo:

```csharp
Presentar("Yoda", 900);

static void Presentar(string nombre, int edad)
{
    Console.WriteLine($"{nombre} tiene {edad} años.");
}
```

Los argumentos se asignan **en orden**: el primero al primer parámetro, el segundo al segundo. Deben coincidir en cantidad y tipo:

```csharp
Presentar(900, "Yoda");   // error CS1503: no se puede convertir de 'int' a 'string'
Presentar("Yoda");        // error CS7036: no se ha dado ningún argumento para el parámetro 'edad'
```

### Parámetros y argumentos, de un vistazo

```text
Definición:   static void Presentar(string nombre, int edad)
                                     ▲              ▲
                                     │              │       parámetros (marcadores de lugar)
Llamada:      Presentar(          "Yoda",          900);
                                  argumentos (valores reales)
```

### Ámbito: los parámetros y las variables locales solo existen dentro del método

```csharp
static void Mostrar(string mensaje)
{
    string copia = mensaje.ToUpper();
    Console.WriteLine(copia);
}

Console.WriteLine(mensaje);   // error CS0103: The name 'mensaje' does not exist in the current context
Console.WriteLine(copia);     // error CS0103
```

Cada método es una "caja cerrada": lo que se declara dentro no se ve desde afuera, y viceversa (salvo en las funciones locales, ver "Para profundizar"). Esto es bueno: puedes usar el nombre `total` en diez métodos distintos sin que choquen.

### Por qué los parámetros reciben copias

Al llamar a un método, el valor de cada argumento se **copia** en el parámetro. Si el método modifica el parámetro, el original no cambia:

```csharp
int puntos = 10;
Duplicar(puntos);
Console.WriteLine(puntos);   // 10

static void Duplicar(int valor)
{
    valor *= 2;               // modifica la copia local
}
```

Con tipos de referencia se copia la **referencia**, así que el método sí puede modificar el objeto compartido (ver [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md)). Para que el método devuelva un resultado, se usa `return`, que se ve en [Valores de retorno y parámetros out](03-Valores%20de%20retorno%20y%20parametros%20out.md).

### Métodos que ya conoces, ahora con nombre técnico

| Llamada | Tipo de método |
| --- | --- |
| `Console.WriteLine("x")` | Método **estático** de la clase `Console`: se llama sobre la clase. |
| `Math.Max(3, 7)` | Método estático de la clase `Math`. |
| `texto.ToUpper()` | Método **de instancia**: se llama sobre un valor concreto (`texto`). |
| `int.Parse("5")` | Método estático del tipo `int`. |

-----

## Ejemplo completo

Un programa que imprime un recibo, dividido en métodos con una sola responsabilidad cada uno:

```csharp
ImprimirEncabezado("Cafetería Central");
ImprimirLinea("Café americano", 2, 8.50m);
ImprimirLinea("Croissant", 1, 6.00m);
ImprimirLinea("Jugo de naranja", 1, 9.90m);
ImprimirSeparador();
Console.WriteLine("Gracias por su compra");

static void ImprimirEncabezado(string negocio)
{
    ImprimirSeparador();
    Console.WriteLine(negocio.ToUpper());
    Console.WriteLine($"Fecha: {DateTime.Now:dd/MM/yyyy}");
    ImprimirSeparador();
}

static void ImprimirLinea(string producto, int cantidad, decimal precioUnitario)
{
    decimal subtotal = cantidad * precioUnitario;
    Console.WriteLine($"{producto,-18} {cantidad,2} x {precioUnitario,6:N2} = {subtotal,7:N2}");
}

static void ImprimirSeparador()
{
    Console.WriteLine(new string('-', 42));
}
```

Salida (la fecha varía):

```text
------------------------------------------
CAFETERÍA CENTRAL
Fecha: 02/10/2026
------------------------------------------
Café americano      2 x   8.50 =   17.00
Croissant           1 x   6.00 =    6.00
Jugo de naranja     1 x   9.90 =    9.90
------------------------------------------
Gracias por su compra
```

Si mañana cambia el ancho del separador, se modifica en **un solo lugar**. Y los métodos se llaman entre sí: `ImprimirEncabezado` usa `ImprimirSeparador`.

-----

## Errores comunes

**1. Olvidar los paréntesis al llamar.**
Qué pasa: `ImprimirSeparador;` da `error CS0201: Only assignment, call, increment, decrement, await, and new object expressions can be used as a statement`.
Por qué: sin paréntesis no hay llamada, solo el nombre del método.
Arreglo: `ImprimirSeparador();`.

**2. Argumentos en distinto orden o tipo.**
Qué pasa: `error CS1503: Argument 1: cannot convert from 'int' to 'string'`.
Por qué: los argumentos se asignan por posición.
Arreglo: respeta el orden de la definición, o usa argumentos con nombre (siguiente lección).

**3. Faltan argumentos.**
Qué pasa: `error CS7036: There is no argument given that corresponds to the required parameter 'edad'`.
Por qué: todos los parámetros sin valor por defecto son obligatorios.
Arreglo: pasa todos los argumentos o define un valor por defecto.

**4. Usar un parámetro fuera de su método.**
Qué pasa: `error CS0103: The name 'mensaje' does not exist in the current context`.
Por qué: el ámbito de un parámetro es su método.
Arreglo: haz que el método devuelva el valor con `return` y guárdalo en una variable afuera.

**5. Llamar a un método de instancia desde `Main` estático.**
Qué pasa: `error CS0120: An object reference is required for the non-static field, method, or property 'Program.Saludar()'`.
Por qué: `Main` es `static` y no tiene un objeto sobre el cual llamar al método.
Arreglo: marca el método como `static` (o crea un objeto de la clase, como se verá en POO).

**6. Definir un método dentro de otro sin querer.**
Qué pasa: con un `Main` explícito, pegar un método dentro de las llaves de `Main` crea una función local; si además repites nombres, aparecen errores confusos de ámbito.
Por qué: las llaves mal cerradas cambian dónde vive el método.
Arreglo: formatea el documento (Ctrl + K, Ctrl + D) y revisa el cierre de llaves.

-----

## Según la versión de C#

* **C# 7:** funciones locales (métodos declarados dentro de otro método).
* **C# 8:** funciones locales `static`, que no pueden capturar variables del método que las contiene.
* **C# 9:** top-level statements: los métodos escritos en `Program.cs` sin clase son funciones locales.

-----

## Cuándo sí y cuándo no

**Extrae un método cuando:**

* Un bloque de código se repite (o se va a repetir).
* Un bloque hace algo que puedes describir con un nombre claro: `ValidarCorreo`, `CalcularImpuesto`.
* Un método supera unas 20-30 líneas o mezcla varias responsabilidades.

**No extraigas un método cuando:**

* Solo sería una línea trivial usada una vez y el nombre no aporta claridad.

**Buenas prácticas:**

* Un método, una responsabilidad.
* Pocos parámetros: con más de 3 o 4, considera agrupar los datos en un objeto.
* El nombre debe decir **qué hace**, no cómo: `ObtenerClientesActivos` y no `RecorrerListaYFiltrar`.

-----

## Resumen en 5 líneas

1. Un método es un bloque de código con nombre; se define una vez y se llama muchas veces.
2. Llamar = nombre + paréntesis: `Saludar("Ana");`. Sin paréntesis no se ejecuta.
3. Parámetros (en la definición) reciben argumentos (en la llamada), en orden y con tipos compatibles.
4. Los parámetros y las variables locales solo existen dentro de su método.
5. `void` = no devuelve nada; nombres en PascalCase y con verbo.

-----

## Para profundizar

<details>
<summary>Funciones locales</summary>

Un método puede contener otros métodos, llamados **funciones locales**. Son útiles para ayudantes que solo tienen sentido dentro de un método:

```csharp
static double PromedioSinExtremos(int[] notas)
{
    return Sumar() / (notas.Length - 2.0);

    double Sumar()    // función local: puede usar 'notas' del método que la contiene
    {
        int total = 0;
        foreach (int n in notas) total += n;
        return total - notas.Max() - notas.Min();
    }
}
```

Si la marcas como `static`, no puede capturar variables externas, lo que evita dependencias ocultas.

</details>

<details>
<summary>Qué pasa en memoria al llamar un método</summary>

Cada llamada apila un nuevo *frame* en el stack con los parámetros, las variables locales y la dirección de retorno. Cuando el método termina, su frame se desapila. Por eso las variables locales "desaparecen" al salir del método, y por eso una recursión sin fin termina con `StackOverflowException`: el stack se llena de frames. Ver [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un método es un bloque de código reutilizable con un nombre. Se define con un tipo de retorno (o `void`), un nombre y una lista de parámetros, y se llama con su nombre y paréntesis, pasándole argumentos. En C# todos los métodos pertenecen a una clase o a otro tipo, por eso no se habla de "funciones sueltas".

### Respuesta ampliada (semi-senior)

La firma de un método está formada por su nombre y los tipos y modificadores (`ref`/`out`/`in`) de sus parámetros; el tipo de retorno no forma parte de ella a efectos de sobrecarga. Por defecto, los argumentos se pasan por valor: los tipos de valor se copian y, en los tipos de referencia, se copia la referencia. Los métodos `static` pertenecen al tipo y no tienen acceso a `this`. Con top-level statements, los métodos de `Program.cs` se compilan como funciones locales del punto de entrada. Los buenos métodos son cortos, tienen una sola responsabilidad y pocos parámetros, lo que los hace fáciles de probar.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre parámetro y argumento?**
El parámetro es la variable declarada en la definición; el argumento es el valor concreto que se pasa en la llamada.

**2. ¿Método o función?**
En C#, toda función pertenece a un tipo, así que técnicamente son métodos. Las "funciones locales" son métodos declarados dentro de otro.

**3. ¿Por qué no puedo llamar a un método no estático desde `Main`?**
Porque `Main` es `static`: no hay una instancia (`this`) sobre la cual invocar un método de instancia.

-----

## Práctica

**Ejercicio 1.** Escribe un método `DibujarRectangulo(int ancho, int alto, char simbolo)` que dibuje un rectángulo con ese carácter. Llámalo con `(5, 3, '*')` y `(8, 2, '#')`.

<details>
<summary>Solución</summary>

```csharp
DibujarRectangulo(5, 3, '*');
Console.WriteLine();
DibujarRectangulo(8, 2, '#');

static void DibujarRectangulo(int ancho, int alto, char simbolo)
{
    for (int fila = 0; fila < alto; fila++)
    {
        Console.WriteLine(new string(simbolo, ancho));
    }
}
```

</details>

**Ejercicio 2.** Este programa no compila. Encuentra los dos errores.

```csharp
MostrarDoble(4)

static void MostrarDoble(int numero)
{
    int doble = numero * 2;
}

Console.WriteLine(doble);
```

<details>
<summary>Solución</summary>

1. Falta el `;` después de `MostrarDoble(4)` (CS1002).
2. `doble` es una variable local de `MostrarDoble`; no existe fuera (CS0103).

Además, hay un problema de diseño: el método calcula el doble pero no lo muestra ni lo devuelve. Versión corregida:

```csharp
MostrarDoble(4);

static void MostrarDoble(int numero)
{
    int doble = numero * 2;
    Console.WriteLine(doble);
}
```

</details>

-----

## Siguiente lección

[Parámetros opcionales, nombrados y sobrecarga](02-Parametros%20opcionales%20nombrados%20y%20sobrecarga.md)
