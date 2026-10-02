# Tipos de valor y de referencia

## En una frase

Una variable de **tipo de valor** (`int`, `double`, `bool`, `struct`) contiene el dato en sí; una variable de **tipo de referencia** (`class`, `string`, arrays) contiene una **dirección** que apunta a un objeto guardado en el *heap*, y el recolector de basura libera ese objeto cuando nadie lo referencia.

-----

## Antes de empezar

Conviene que ya sepas:

* Declarar variables y conocer los tipos básicos, como en [Variables y tipos de datos](01-Variables%20y%20tipos%20de%20datos.md).
* Que existe algo llamado clase y que se crean objetos con `new`. No hace falta más: las clases se estudian en [Clases y objetos](../04-poo/01-Clases%20y%20objetos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Tipo de valor:** la variable guarda el dato directamente. Copiarla copia el dato.
* **Tipo de referencia:** la variable guarda una referencia a un objeto. Copiarla copia la referencia, no el objeto.
* **Referencia:** un valor que identifica dónde está un objeto en memoria (parecido a una dirección).
* **Stack (pila):** zona de memoria rápida donde se guardan las variables locales y los datos de cada llamada a método.
* **Heap (montón):** zona de memoria donde viven los objetos creados con `new`.
* **GC (Garbage Collector):** el componente que libera automáticamente los objetos del heap que ya no se usan.
* **`null`:** el valor de una referencia que no apunta a ningún objeto.

-----

## El problema

Mira este código y adivina qué imprime:

```csharp
int a = 10;
int b = a;
b = 99;
Console.WriteLine(a);        // ¿10 o 99?

int[] x = { 10 };
int[] y = x;
y[0] = 99;
Console.WriteLine(x[0]);     // ¿10 o 99?
```

Imprime `10` y luego `99`. Las dos parecen "copiar y modificar", pero se comportan distinto. Si no entiendes por qué, vas a tener bugs en los que "un objeto cambió solo" porque otra parte del código lo modificó a través de otra variable. Es una de las fuentes de errores más comunes en C#.

-----

## Cómo funciona

### Tipos de valor: la variable ES el dato

```csharp
int a = 10;
int b = a;     // se COPIA el valor 10 dentro de b
b = 99;        // solo cambia b
```

```text
a ┌────┐      b ┌────┐
  │ 10 │        │ 99 │      dos cajas independientes
  └────┘        └────┘
```

Son tipos de valor:

* Todos los numéricos (`int`, `long`, `double`, `decimal`...), `bool` y `char`.
* Los `struct` (incluidos `DateTime`, `Guid`, `TimeSpan`).
* Los `enum`.
* Las tuplas `(int, string)`.

### Tipos de referencia: la variable APUNTA al dato

```csharp
int[] x = { 10 };
int[] y = x;    // se COPIA la referencia: x e y apuntan al MISMO array
y[0] = 99;      // se modifica el array compartido
```

```text
x ┌──────┐
  │ ref ─┼──┐
  └──────┘  │      heap
            ├──►  ┌──────┐
y ┌──────┐  │     │ [99] │   un solo array
  │ ref ─┼──┘     └──────┘
  └──────┘
```

Son tipos de referencia:

* Las `class` (las tuyas y las de .NET: `List<T>`, `Random`, `StringBuilder`...).
* `string` y `object`.
* Los arrays (`int[]`, `string[]`), aunque contengan tipos de valor.
* Las `interface`, los `delegate` y los `record` (salvo `record struct`).

### Lo mismo con una clase

```csharp
class Persona
{
    public string Nombre = "";
}
```

```csharp
var p1 = new Persona { Nombre = "Ana" };
var p2 = p1;               // p2 apunta al MISMO objeto
p2.Nombre = "Luis";

Console.WriteLine(p1.Nombre);   // Luis

var p3 = new Persona { Nombre = "Luis" };
Console.WriteLine(p1 == p3);    // False: son objetos distintos, aunque tengan el mismo contenido
```

`==` entre clases compara **referencias** (si apuntan al mismo objeto), no el contenido. Hay excepciones, como `string` y los `record`, que se explican más adelante.

### `null`: una referencia que no apunta a nada

Una variable de tipo referencia puede valer `null`:

```csharp
Persona? p = null;
Console.WriteLine(p.Nombre);   // NullReferenceException al ejecutar
```

El `?` en `Persona?` indica "puede ser null". Con `<Nullable>enable</Nullable>`, el compilador te advierte (CS8602) antes de ejecutar. Un tipo de valor normal **no** puede ser `null` (`int x = null;` no compila); para eso existe `int?`.

### `string`: referencia que se comporta como valor

`string` es un tipo de referencia, pero tiene dos particularidades que lo hacen sentir como un valor:

1. **Es inmutable:** ningún método modifica el string original; todos devuelven uno nuevo.
2. **`==` compara el contenido**, no la referencia.

```csharp
string s1 = "hola";
string s2 = s1;
s2 = s2.ToUpper();         // crea un string NUEVO y s2 apunta a él

Console.WriteLine(s1);     // hola (no cambió)
Console.WriteLine("ab" + "c" == "abc");   // True: compara el contenido
```

Se profundiza en [Texto: char y string](05-Texto%20char%20y%20string.md).

### Stack y heap: dónde vive cada cosa

La explicación habitual es "los tipos de valor van en el stack y los de referencia en el heap". Sirve como primera aproximación, pero **no es exacta**. La regla correcta es:

* Las **variables locales** de un método viven en el **stack** (dentro del *frame* de ese método).
* Los **objetos** creados con `new` a partir de una clase, y los arrays, viven en el **heap**.

```csharp
void Ejemplo()
{
    int edad = 30;                    // local de valor: el 30 está en el stack
    var p = new Persona();            // local de referencia: la REFERENCIA está en el stack,
                                      // el objeto Persona está en el heap
}
```

```text
STACK (frame de Ejemplo)          HEAP
┌───────────────────┐
│ edad = 30         │
│ p    = ref ───────┼──────►  ┌──────────────────┐
└───────────────────┘          │ Persona          │
                               │  Nombre = ""     │
                               └──────────────────┘
```

Por eso la regla simplificada falla: un `int` que es **campo** de una clase vive **dentro del objeto, en el heap**. Lo que define el tipo de valor no es *dónde* vive, sino que **se copia por valor**.

### El stack: LIFO

El stack funciona como una pila de platos: **el último en entrar es el primero en salir** (LIFO, *Last In, First Out*). Cada vez que llamas a un método, se apila un *frame* con sus parámetros, sus variables locales y la dirección de retorno (a dónde volver cuando termine). Cuando el método termina, su frame se desapila y esas variables desaparecen. No hace falta recolector de basura para el stack: se limpia solo y es muy rápido.

```text
Main() llama a A(), A() llama a B():

│ B: variables de B  │  ← tope (se quita primero cuando B termina)
│ A: variables de A  │
│ Main: variables    │
└────────────────────┘
```

### El heap y el recolector de basura (GC)

Los objetos del heap no desaparecen cuando termina un método, porque otras variables pueden seguir apuntándolos. ¿Quién los libera? El **GC**:

1. Creas un objeto con `new` → se reserva espacio en el heap.
2. Mientras alguna referencia **alcanzable** apunte al objeto, sigue vivo.
3. Cuando ninguna referencia alcanzable lo apunta, el objeto es "basura".
4. En algún momento (no sabes cuándo), el GC lo detecta y libera su memoria.

```csharp
var p = new Persona();   // objeto A en el heap
p = new Persona();       // p apunta a un objeto B; A ya no tiene referencias → candidato a GC
p = null;                // B tampoco → candidato a GC
```

No llamas a `free()` ni a `delete` como en C o C++.

### Variables capturadas en lambdas

Una variable local normalmente muere cuando termina el método. Pero si una **lambda** la usa, la variable puede "vivir más":

```csharp
int contador = 0;
Action incrementar = () => contador++;   // la lambda CAPTURA la variable contador

incrementar();
incrementar();
Console.WriteLine(contador);             // 2
```

El compilador mueve `contador` a un objeto oculto en el heap (una *clausura* o *closure*) para que la lambda pueda seguir usándola. La lambda captura la **variable**, no una copia de su valor. Se retoma en [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).

-----

## Ejemplo completo

```csharp
// Tipo de valor: struct
var p1 = new PuntoValor { X = 1 };
var p2 = p1;          // copia del dato
p2.X = 50;
Console.WriteLine($"struct → p1.X = {p1.X}, p2.X = {p2.X}");   // 1, 50

// Tipo de referencia: class
var r1 = new PuntoReferencia { X = 1 };
var r2 = r1;          // copia de la referencia
r2.X = 50;
Console.WriteLine($"class  → r1.X = {r1.X}, r2.X = {r2.X}");   // 50, 50

// Pasar a un método
Duplicar(p1);
DuplicarRef(r1);
Console.WriteLine($"Después de los métodos → p1.X = {p1.X}, r1.X = {r1.X}");   // 1, 100

static void Duplicar(PuntoValor p) => p.X *= 2;            // modifica una COPIA
static void DuplicarRef(PuntoReferencia r) => r.X *= 2;    // modifica el objeto compartido

struct PuntoValor { public int X; }
class PuntoReferencia { public int X; }
```

Salida:

```text
struct → p1.X = 1, p2.X = 50
class  → r1.X = 50, r2.X = 50
Después de los métodos → p1.X = 1, r1.X = 100
```

`struct` y `class` tienen la misma sintaxis, pero se copian de forma opuesta. Al pasar argumentos a un método ocurre lo mismo que al asignar.

-----

## Errores comunes

**1. Creer que asignar un objeto lo copia.**
Qué pasa: modificas `copia` y el "original" también cambia.
Por qué: con tipos de referencia, `var copia = original;` copia la referencia, no el objeto.
Arreglo: si necesitas una copia, créala explícitamente (un constructor que copie, `with` en records o un método `Clonar`).

**2. Usar una referencia `null`.**
Qué pasa: `System.NullReferenceException: Object reference not set to an instance of an object.`
Por qué: llamaste a un miembro (`.Nombre`, `.Length`) sobre una variable que vale `null`.
Arreglo: activa `Nullable`, atiende las advertencias CS8602 y comprueba con `if (x is not null)` o usa `?.`.

**3. Esperar que `==` compare el contenido de dos objetos de una clase.**
Qué pasa: `p1 == p3` da `False` aunque tengan los mismos datos.
Por qué: para las clases, `==` compara referencias por defecto.
Arreglo: compara las propiedades, sobrescribe `Equals` o usa un `record`. Se ve en [La clase Object](../04-poo/10-La%20clase%20Object.md).

**4. Modificar un `struct` dentro de un método y esperar ver el cambio afuera.**
Qué pasa: el cambio "se pierde".
Por qué: el método recibió una copia.
Arreglo: devuelve el nuevo valor o pasa el parámetro con `ref`. Se ve en [Valores de retorno y parámetros out](../03-metodos/03-Valores%20de%20retorno%20y%20parametros%20out.md).

-----

## Según la versión de C#

* **C# 2:** tipos de valor que aceptan nulos: `int?`.
* **C# 8:** tipos de referencia que aceptan nulos (`string?`) y advertencias del compilador sobre `null`.
* **C# 9:** `record` (referencia con igualdad por contenido).
* **C# 10:** `record struct` (tipo de valor con las ventajas de los records).

-----

## Cuándo sí y cuándo no

**Define un tipo de valor (`struct`) cuando:**

* Representa un valor pequeño (unos pocos campos), como una coordenada o un rango de fechas.
* Es lógicamente inmutable y se compara por contenido.

**Define un tipo de referencia (`class`) cuando:**

* Tiene identidad propia (un cliente, un pedido) o es grande.
* Lo van a compartir y modificar varias partes del código.

En la práctica, **casi todo lo que escribas serán clases**. `struct` se reserva para casos concretos.

-----

## Resumen en 5 líneas

1. Tipo de valor: la variable contiene el dato; asignar copia el dato.
2. Tipo de referencia: la variable contiene una referencia; asignar copia la referencia y ambas apuntan al mismo objeto.
3. Las variables locales viven en el stack; los objetos creados con `new` viven en el heap.
4. El GC libera los objetos del heap que ya no son alcanzables.
5. `string` es de referencia, pero inmutable y con `==` por contenido.

-----

## Para profundizar

<details>
<summary>GC generacional</summary>

El GC de .NET divide el heap en **generaciones**: los objetos nuevos nacen en la generación 0, que se revisa con mucha frecuencia porque la mayoría de los objetos muere joven. Los que sobreviven pasan a la 1 y luego a la 2, que se revisan menos. Los objetos de más de 85.000 bytes van directo al **LOH** (*Large Object Heap*). Durante una recolección, el GC puede **compactar** el heap y mover objetos, y por eso no trabajas con direcciones de memoria reales sino con referencias.

</details>

<details>
<summary>Finalizadores, IDisposable y using</summary>

El GC libera **memoria**, pero no cierra archivos ni conexiones a bases de datos. Para eso existe la interfaz `IDisposable` y la sentencia `using`, que llama a `Dispose()` en cuanto termina el bloque:

```csharp
using var archivo = new StreamWriter("log.txt");
archivo.WriteLine("Hola");
// al salir del ámbito se llama a archivo.Dispose() y el archivo se cierra
```

Un **finalizador** (`~Clase() { }`) es un método que el GC ejecuta antes de liberar un objeto. Es un último recurso: no sabes cuándo se ejecutará, retrasa la liberación del objeto y tiene costo. En código moderno casi nunca se escribe; se usa `IDisposable`.

</details>

<details>
<summary>Boxing: cuando un valor se convierte en objeto</summary>

Si asignas un tipo de valor a una variable `object` (o a una interfaz), el runtime crea un objeto en el heap y copia el valor dentro. Eso es **boxing**:

```csharp
int n = 5;
object o = n;      // boxing: nueva caja en el heap con una copia de 5
int m = (int)o;    // unboxing: copia el valor de vuelta
```

Cada boxing reserva memoria. En bucles muy calientes puede afectar el rendimiento; los genéricos (`List<int>` en lugar de `ArrayList`) existen en parte para evitarlo. Se retoma en [La clase Object](../04-poo/10-La%20clase%20Object.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los tipos de valor, como `int`, `bool` o los `struct`, guardan el dato directamente, y al asignarlos se copia el valor. Los tipos de referencia, como las clases, los arrays o `string`, guardan una referencia a un objeto en el heap; al asignarlos se copia la referencia y ambas variables apuntan al mismo objeto. El GC libera los objetos que ya no se referencian.

### Respuesta ampliada (semi-senior)

La diferencia está en la semántica de copia, no estrictamente en la ubicación: las locales viven en el stack, pero un campo `int` de una clase vive dentro del objeto en el heap, y una local capturada por una lambda se mueve a una clausura en el heap. Los tipos de valor derivan de `System.ValueType`; asignarlos a `object` o a una interfaz produce boxing. El GC es generacional (gen 0, 1 y 2, más el LOH), recolecta objetos inalcanzables y puede compactar el heap. Los recursos no administrados se liberan de forma determinista con `IDisposable` y `using`, no con finalizadores. `string` es de referencia pero inmutable y sobrecarga `==` para comparar contenido.

### Preguntas frecuentes de seguimiento

**1. ¿Los tipos de valor siempre están en el stack?**
No. Las variables locales sí (salvo que las capture una lambda o un método `async`), pero un tipo de valor que es campo de una clase o elemento de un array vive en el heap junto con el objeto que lo contiene.

**2. ¿`string` es de valor o de referencia?**
De referencia. Se comporta como valor porque es inmutable y su `==` compara el contenido.

**3. ¿Cuándo se ejecuta el GC?**
No es determinista: cuando el runtime lo decide, normalmente por presión de memoria en la generación 0. Se puede forzar con `GC.Collect()`, pero casi nunca conviene hacerlo.

**4. ¿Qué es boxing y por qué importa?**
Convertir un tipo de valor en un objeto del heap. Cada boxing reserva memoria y genera trabajo para el GC.

-----

## Práctica

**Ejercicio 1.** ¿Qué imprime este código?

```csharp
var lista1 = new List<int> { 1, 2 };
var lista2 = lista1;
lista2.Add(3);

int n1 = 5;
int n2 = n1;
n2++;

Console.WriteLine($"{lista1.Count} {n1}");
```

<details>
<summary>Solución</summary>

`3 5`. `List<int>` es una clase (referencia): `lista1` y `lista2` son la misma lista. `int` es de valor: `n2` es una copia independiente.

</details>

**Ejercicio 2.** Explica por qué este código **no** modifica la palabra:

```csharp
string palabra = "hola";
palabra.ToUpper();
Console.WriteLine(palabra);   // hola
```

<details>
<summary>Solución</summary>

Los strings son inmutables: `ToUpper()` devuelve un **nuevo** string y no cambia el original. Como no se guardó el resultado, se pierde. Arreglo: `palabra = palabra.ToUpper();`.

</details>

-----

## Siguiente lección

[Conversiones de tipos](03-Conversiones%20de%20tipos.md)
