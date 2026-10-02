# Restricciones y varianza

## En una frase

Las **restricciones** (`where T : ...`) limitan qué tipos se pueden usar como `T` y, a cambio, te permiten usar sus miembros (`new T()`, `CompareTo`, propiedades de una clase base); la **varianza** (`out T`, `in T`) permite que una interfaz o delegado genérico se convierta a otro con un tipo más general o más específico.

-----

## Antes de empezar

Conviene que ya sepas:

* Clases, métodos e interfaces genéricas y `default`, de [Genéricos](04-Genericos.md).
* Upcasting, herencia e interfaces como `IComparable<T>`, de [Polimorfismo y casting](../04-poo/09-Polimorfismo%20y%20casting.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Restricción genérica (*constraint*):** condición que debe cumplir un argumento de tipo, declarada con `where`.
* **Covarianza (`out`):** permite usar `I<Derivado>` donde se espera `I<Base>`. Solo para tipos que se **devuelven**.
* **Contravarianza (`in`):** permite usar `I<Base>` donde se espera `I<Derivado>`. Solo para tipos que se **reciben**.
* **Invarianza:** solo se acepta exactamente el mismo tipo.
* **Más derivado / menos derivado:** más específico (`Perro`) / más general (`Animal`).

-----

## El problema

Escribes un método genérico para obtener el mayor de dos valores:

```csharp
static T Mayor<T>(T a, T b) => a.CompareTo(b) > 0 ? a : b;   // error CS1061: 'T' no contiene 'CompareTo'
```

El compilador tiene razón: `T` puede ser **cualquier** tipo, y no todos se pueden comparar. Necesitas decirle "`T` será siempre algo comparable".

Y un segundo problema: tienes un método que imprime cualquier lista de animales:

```csharp
static void ImprimirTodos(List<Animal> animales) { /* ... */ }

List<Perro> perros = ObtenerPerros();
ImprimirTodos(perros);   // error CS1503: no se puede convertir de 'List<Perro>' a 'List<Animal>'
```

Un perro **es un** animal, pero una lista de perros **no es** una lista de animales. ¿Por qué? ¿Y cómo se escribe ese método para que funcione?

-----

## Cómo funciona

### Parte 1: restricciones

Se declaran con `where` después de la firma:

```csharp
static T Mayor<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) > 0 ? a : b;

Console.WriteLine(Mayor(3, 7));            // 7
Console.WriteLine(Mayor("pera", "mango")); // pera
Mayor(new object(), new object());         // error CS0311: 'object' no cumple la restricción
```

Ahora el compilador sabe que todo `T` implementa `IComparable<T>`, así que permite llamar a `CompareTo`. A cambio, rechaza tipos que no cumplen.

### Los tipos de restricción

| Restricción | `T` debe ser... | Te permite... |
| --- | --- | --- |
| `where T : struct` | Un tipo de valor que no acepta null (`int`, `DateTime`, structs) | Usar `T?` como `Nullable<T>` |
| `where T : class` | Un tipo de referencia | Usar `null` y comparar referencias |
| `where T : class?` | Un tipo de referencia que puede ser anulable | — |
| `where T : notnull` | Un tipo que no acepta null (de valor o de referencia) | Usarlo como clave de diccionario sin advertencias |
| `where T : unmanaged` | Un tipo sin referencias (primitivos, structs simples) | Código de bajo nivel, punteros |
| `where T : new()` | Un tipo con constructor público sin parámetros | `new T()` |
| `where T : ClaseBase` | Esa clase o una derivada | Usar los miembros de `ClaseBase` |
| `where T : IInterfaz` | Un tipo que implementa la interfaz | Usar los miembros de la interfaz |
| `where T : U` | `U` o un tipo derivado de otro parámetro `U` | Relacionar dos parámetros de tipo |
| `where T : Enum` / `Delegate` | Un enum / un delegado | Métodos genéricos sobre enums o delegados |

### Combinar restricciones

```csharp
class Repositorio<T> where T : Entidad, IValidable, new()
{
    public T CrearVacio() => new T();          // gracias a new()
    public bool Guardar(T item) => item.Validar() && item.Id >= 0;   // IValidable y Entidad
}
```

Reglas de orden:

1. Primero `class`, `struct`, `notnull`, `unmanaged` **o** una clase base (solo una de ellas).
2. Después las interfaces (todas las que quieras).
3. Al final `new()`.

Además: `struct` y `new()` no se combinan (todo struct ya tiene constructor sin parámetros), y `class` y `struct` son excluyentes.

Con varios parámetros de tipo, cada uno tiene su propio `where`:

```csharp
interface IPipeline<TEntrada, TSalida>
    where TEntrada : SolicitudBase
    where TSalida : IDisposable, new()
{
    TSalida Ejecutar(TEntrada solicitud);
}
```

### Por qué `List<Perro>` no es una `List<Animal>`

Imagina que el compilador lo permitiera:

```csharp
List<Perro> perros = new() { new Perro() };
List<Animal> animales = perros;     // supongamos que compila...
animales.Add(new Gato());           // ...y ahora hay un GATO dentro de una lista de PERROS
Perro p = perros[1];                // 💥 ¿qué devuelve esto?
```

Como `List<T>` permite **leer y escribir**, aceptar esa conversión rompería la seguridad de tipos. Por eso `List<T>` es **invariante**.

### Parte 2: covarianza (`out T`)

Si una interfaz solo **devuelve** `T` (nunca lo recibe), no hay forma de "meter un gato". Entonces la conversión es segura, y se marca con `out`:

```csharp
// .NET la declara así: public interface IEnumerable<out T>
IEnumerable<Perro> perros = new List<Perro> { new Perro() };
IEnumerable<Animal> animales = perros;    // ✅ covarianza: Perro → Animal
```

`IEnumerable<T>` solo permite leer elementos, así que tratar una secuencia de perros como una secuencia de animales es siempre seguro. Por eso la solución al problema del principio es:

```csharp
static void ImprimirTodos(IEnumerable<Animal> animales)
{
    foreach (var a in animales) Console.WriteLine(a.Nombre);
}

ImprimirTodos(perros);   // ✅
```

Declarar tu propia interfaz covariante:

```csharp
interface IProductor<out T>
{
    T Producir();            // T solo aparece como salida
}

class FabricaDePerros : IProductor<Perro>
{
    public Perro Producir() => new Perro();
}

IProductor<Animal> fabrica = new FabricaDePerros();   // ✅
Animal a = fabrica.Producir();                         // produce un Perro, visto como Animal
```

### Contravarianza (`in T`)

Si una interfaz solo **recibe** `T` (nunca lo devuelve), la conversión segura va en sentido **contrario**: algo que sabe manejar cualquier `Animal` sabe manejar un `Perro`.

```csharp
interface IConsumidor<in T>
{
    void Consumir(T item);   // T solo aparece como entrada
}

class Veterinario : IConsumidor<Animal>
{
    public void Consumir(Animal a) => Console.WriteLine($"Atendiendo a {a.Nombre}");
}

IConsumidor<Perro> paraPerros = new Veterinario();   // ✅ contravarianza: Animal → Perro
paraPerros.Consumir(new Perro());
```

Ejemplos en .NET:

| Tipo | Declaración | Varianza |
| --- | --- | --- |
| `IEnumerable<T>` | `IEnumerable<out T>` | Covariante |
| `IReadOnlyList<T>` | `IReadOnlyList<out T>` | Covariante |
| `IComparer<T>` | `IComparer<in T>` | Contravariante |
| `Action<T>` | `Action<in T>` | Contravariante |
| `Func<T, TResult>` | `Func<in T, out TResult>` | Contra en la entrada, co en la salida |
| `List<T>`, `IList<T>` | sin modificador | Invariante |

### Reglas de la varianza

* Solo se aplica a **interfaces y delegados** genéricos, no a clases (`List<T>` nunca es covariante).
* Solo funciona con **tipos de referencia**: `IEnumerable<int>` no se convierte en `IEnumerable<object>`, porque `int` → `object` requiere boxing.
* `out T`: `T` solo puede aparecer en posiciones de salida (retornos, propiedades de solo lectura).
* `in T`: `T` solo puede aparecer en posiciones de entrada (parámetros).

Truco para recordarlo: **`out` = sale de la interfaz (productor); `in` = entra a la interfaz (consumidor).**

```text
Jerarquía:        Animal  ◄── Perro          (un Perro ES un Animal)

COVARIANZA (out): la conversión va en el MISMO sentido que la herencia
   IEnumerable<Perro>  ─────────►  IEnumerable<Animal>      ✔ "lo que sale es un Perro, y todo Perro es Animal"

CONTRAVARIANZA (in): la conversión va en sentido CONTRARIO
   IComparer<Animal>   ─────────►  IComparer<Perro>         ✔ "si sabe comparar animales, sabe comparar perros"

INVARIANZA (List<T>): ninguna conversión
   List<Perro>         ────✘────   List<Animal>             ✘ "podrías meter un Gato"
```

-----

## Ejemplo completo

```csharp
// Restricciones: un pipeline que crea su resultado y lo libera al final
var pipeline = new Pipeline<SolicitudLogin, ResultadoLogin>();
using (var resultado = pipeline.Ejecutar(new SolicitudLogin { Usuario = "ana" }))
{
    Console.WriteLine($"Token: {resultado.Token}");
}

// Restricción con interfaz: buscar el máximo de cualquier secuencia comparable
Console.WriteLine(Maximo(new[] { 4, 9, 2 }));
Console.WriteLine(Maximo(new[] { "kiwi", "uva", "pera" }));

// Covarianza: una lista de perros se procesa como IEnumerable<Animal>
List<Perro> perros = new() { new Perro("Firulais"), new Perro("Toby") };
ImprimirTodos(perros);

// Contravarianza: un comparador de animales sirve para ordenar perros
IComparer<Animal> porNombre = new ComparadorPorNombre();
perros.Sort(porNombre);
Console.WriteLine(string.Join(", ", perros.Select(p => p.Nombre)));

static T Maximo<T>(IEnumerable<T> items) where T : IComparable<T>
{
    T? max = default;
    bool primero = true;
    foreach (var item in items)
    {
        if (primero || item.CompareTo(max!) > 0)
        {
            max = item;
            primero = false;
        }
    }
    return primero ? throw new InvalidOperationException("Secuencia vacía") : max!;
}

static void ImprimirTodos(IEnumerable<Animal> animales)
{
    foreach (var a in animales) Console.WriteLine($"- {a.Nombre}");
}

class SolicitudBase { public DateTime Fecha { get; } = DateTime.Now; }
class SolicitudLogin : SolicitudBase { public string Usuario { get; init; } = ""; }

class ResultadoLogin : IDisposable
{
    public string Token { get; set; } = "";
    public void Dispose() => Console.WriteLine("Recursos del resultado liberados");
}

class Pipeline<TEntrada, TSalida>
    where TEntrada : SolicitudBase
    where TSalida : IDisposable, new()
{
    public TSalida Ejecutar(TEntrada solicitud)
    {
        Console.WriteLine($"Procesando solicitud del {solicitud.Fecha:HH:mm}");   // miembro de SolicitudBase
        var salida = new TSalida();                                                // gracias a new()
        if (salida is ResultadoLogin login) login.Token = Guid.NewGuid().ToString()[..8];
        return salida;
    }
}

class Animal
{
    public string Nombre { get; }
    public Animal(string nombre) => Nombre = nombre;
}

class Perro : Animal
{
    public Perro(string nombre) : base(nombre) { }
}

class ComparadorPorNombre : IComparer<Animal>
{
    public int Compare(Animal? x, Animal? y) => string.Compare(x?.Nombre, y?.Nombre, StringComparison.Ordinal);
}
```

Salida (la hora y el token varían):

```text
Procesando solicitud del 10:30
Token: 7f3a9c21
Recursos del resultado liberados
9
uva
- Firulais
- Toby
Firulais, Toby
```

`perros.Sort` espera un `IComparer<Perro>`, y recibe un `IComparer<Animal>`: la contravarianza de `IComparer<in T>` lo permite.

-----

## Errores comunes

**1. Usar miembros que `T` no garantiza.**
Qué pasa: `error CS1061: 'T' does not contain a definition for 'CompareTo'`.
Por qué: sin restricciones, `T` solo tiene los miembros de `object`.
Arreglo: `where T : IComparable<T>` (o la interfaz o clase base que corresponda).

**2. Pasar un tipo que no cumple la restricción.**
Qué pasa: `error CS0311: The type 'object' cannot be used as type parameter 'T'... There is no implicit reference conversion from 'object' to 'System.IComparable<object>'.`
Por qué: el tipo no implementa lo que exige el `where`.
Arreglo: usa un tipo que cumpla o revisa si la restricción es demasiado estricta.

**3. `new T()` sin la restricción `new()`.**
Qué pasa: `error CS0304: Cannot create an instance of the variable type 'T' because it does not have the new() constraint`.
Por qué: no todo tipo tiene un constructor sin parámetros.
Arreglo: agrega `where T : new()` (al final de la lista).

**4. Orden incorrecto de las restricciones.**
Qué pasa: `error CS0401: The new() constraint must be the last constraint specified` o `error CS0449: The 'class', 'struct', 'unmanaged', 'notnull', and 'default' constraints cannot be combined or duplicated, and must be specified first in the constraints list`.
Por qué: la sintaxis exige un orden.
Arreglo: `class`/`struct`/clase base → interfaces → `new()`.

**5. Esperar varianza en clases.**
Qué pasa: `List<Animal> l = new List<Perro>();` da `error CS0029: Cannot implicitly convert type 'List<Perro>' to 'List<Animal>'`.
Por qué: las clases genéricas son invariantes.
Arreglo: acepta `IEnumerable<Animal>` o `IReadOnlyList<Animal>` en tus métodos.

**6. Esperar varianza con tipos de valor.**
Qué pasa: `IEnumerable<object> o = new List<int>();` no compila.
Por qué: la varianza solo funciona entre tipos de referencia.
Arreglo: `numeros.Cast<object>()` o `numeros.Select(n => (object)n)`.

-----

## Según la versión de C#

* **C# 2:** restricciones `struct`, `class`, `new()`, clase base e interfaces.
* **C# 4:** varianza `in` / `out` en interfaces y delegados genéricos.
* **C# 7.3:** restricciones `unmanaged`, `Enum` y `Delegate`.
* **C# 8:** `notnull` y `class?`.
* **C# 11:** miembros `static abstract` en interfaces, que permiten restricciones como `where T : INumber<T>`.
* **C# 13:** `allows ref struct`, para que `T` pueda ser un `ref struct` como `Span<T>`.

-----

## Cuándo sí y cuándo no

**Agrega una restricción cuando:**

* Necesitas usar un miembro concreto de `T` (comparar, crear, leer una propiedad de la base).
* Quieres impedir usos sin sentido (un `Repositorio<int>`).

**Evita restricciones innecesarias:**

* Cada restricción reduce los tipos con los que se puede usar tu código. Pide solo lo que de verdad necesitas.

**Para la varianza:**

* En los parámetros de tus métodos, acepta interfaces covariantes (`IEnumerable<T>`, `IReadOnlyList<T>`) en lugar de `List<T>`: así aceptas más tipos de colecciones y de elementos.
* Al diseñar interfaces genéricas, separa las operaciones de lectura (`out`) y de escritura (`in`) si quieres aprovechar la varianza.

-----

## Resumen en 5 líneas

1. `where T : ...` restringe los tipos posibles y habilita sus miembros: interfaces, clase base, `new()`, `class`, `struct`, `notnull`.
2. Orden: `class`/`struct`/clase base, luego interfaces, al final `new()`.
3. `List<Perro>` no es `List<Animal>`: las clases genéricas son invariantes porque permiten leer y escribir.
4. `out T` (covarianza): una interfaz que solo devuelve `T` acepta tipos más derivados (`IEnumerable<Perro>` → `IEnumerable<Animal>`).
5. `in T` (contravarianza): una que solo recibe `T` acepta tipos más generales (`IComparer<Animal>` → `IComparer<Perro>`).

-----

## Para profundizar

<details>
<summary>Varianza en delegados</summary>

Los delegados `Func` y `Action` también son variantes:

```csharp
Func<Perro> crearPerro = () => new Perro("Rex");
Func<Animal> crearAnimal = crearPerro;           // covarianza en el retorno

Action<Animal> acariciar = a => Console.WriteLine($"Acariciando a {a.Nombre}");
Action<Perro> acariciarPerro = acariciar;        // contravarianza en el parámetro
```

Por eso puedes pasar una lambda que recibe un `Animal` a un método que espera un `Action<Perro>`.

</details>

<details>
<summary>Covarianza de arrays: la excepción histórica</summary>

Los arrays sí son covariantes, aunque sean de lectura y escritura. Es una decisión heredada que el runtime compensa con una verificación en cada escritura:

```csharp
Animal[] animales = new Perro[2];
animales[0] = new Gato("Michi");   // ArrayTypeMismatchException al ejecutar
```

Es justamente el problema que la invarianza de `List<T>` evita al compilar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las restricciones genéricas, con `where`, limitan los tipos que se pueden usar como `T`; por ejemplo, `where T : class`, `where T : new()` o `where T : IComparable<T>`. Así se pueden usar los miembros de ese tipo dentro del código genérico. La covarianza (`out`) y la contravarianza (`in`) permiten convertir interfaces genéricas entre tipos más o menos derivados, como pasar un `IEnumerable<Perro>` donde se espera un `IEnumerable<Animal>`.

### Respuesta ampliada (semi-senior)

Las restricciones amplían lo que el compilador permite sobre `T` (llamar a miembros de interfaces o de una base, `new T()`, `null`) y se verifican en cada instanciación; desde C# 11, con miembros `static abstract`, habilitan operadores y fábricas estáticas (generic math). La varianza es segura solo cuando el parámetro de tipo aparece en una única dirección: `out` en posiciones de salida (covarianza, `IEnumerable<out T>`) e `in` en posiciones de entrada (contravarianza, `IComparer<in T>`). Se aplica a interfaces y delegados, nunca a clases, y solo a conversiones de referencia, por eso no funciona con tipos de valor. Los arrays son covariantes por herencia histórica y lo pagan con `ArrayTypeMismatchException` en ejecución.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `List<string>` no se puede asignar a `List<object>`?**
Porque `List<T>` permite escribir: podrías agregar un `int` a una lista que en realidad es de `string`. Es invariante para mantener la seguridad de tipos.

**2. ¿Por qué `IEnumerable<string>` sí se puede asignar a `IEnumerable<object>`?**
Porque `IEnumerable<out T>` solo permite leer: cada `string` que salga es un `object` válido.

**3. ¿Qué restricción necesitas para hacer `new T()`?**
`where T : new()`, que debe ir al final de la lista de restricciones.

-----

## Práctica

**Ejercicio 1.** Escribe un método `Clonar<T>(int cantidad)` que devuelva una lista con `cantidad` instancias nuevas de `T`. Debe funcionar con cualquier clase que tenga un constructor sin parámetros. ¿Qué restricción necesitas?

<details>
<summary>Solución</summary>

```csharp
var lista = Clonar<Tarea>(3);
Console.WriteLine(lista.Count);   // 3

static List<T> Clonar<T>(int cantidad) where T : new()
{
    var resultado = new List<T>(cantidad);
    for (int i = 0; i < cantidad; i++)
        resultado.Add(new T());
    return resultado;
}

class Tarea { public string Titulo { get; set; } = "Sin título"; }
```

</details>

**Ejercicio 2.** ¿Cuáles de estas líneas compilan? `Perro` deriva de `Animal`.

```csharp
IEnumerable<Animal> a = new List<Perro>();          // (1)
List<Animal> b = new List<Perro>();                 // (2)
IReadOnlyList<Animal> c = new List<Perro>();        // (3)
Action<Perro> d = (Animal x) => { };                // (4)
Func<Animal> e = () => new Perro("Rex");            // (5)
IEnumerable<object> f = new List<int>();            // (6)
```

<details>
<summary>Solución</summary>

1. Sí: `IEnumerable<out T>` es covariante.
2. No: `List<T>` es invariante.
3. Sí: `IReadOnlyList<out T>` es covariante.
4. No: `error CS1661`. Una lambda con tipo de parámetro **explícito** debe coincidir exactamente con el delegado (`Perro`). La contravarianza se aplica entre **delegados**, no a las lambdas: `Action<Animal> x = a => { }; Action<Perro> d = x;` sí compila.
5. Sí: la lambda devuelve un `Perro`, que es un `Animal` (conversión implícita del valor de retorno).
6. No: la varianza no funciona con tipos de valor (`int`).

</details>

-----

## Siguiente lección

Terminaste el módulo de tipos avanzados. Continúa con [Colecciones](../06-colecciones/README.md).
