# Genéricos

## En una frase

Los genéricos permiten escribir clases, interfaces y métodos con un **parámetro de tipo** (`T`) que se decide al usarlos (`List<int>`, `List<string>`), para reutilizar el mismo código con cualquier tipo **sin perder la verificación del compilador** ni pagar boxing.

-----

## Antes de empezar

Conviene que ya sepas:

* Clases, interfaces y métodos, de [POO](../04-poo/README.md).
* Qué es el boxing y por qué `object` pierde la información del tipo, de [La clase Object](../04-poo/10-La%20clase%20Object.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Genérico:** clase, interfaz, método o delegado con uno o más parámetros de tipo.
* **Parámetro de tipo:** el marcador (`T`, `TKey`) que representa un tipo a definir después.
* **Argumento de tipo:** el tipo concreto que se usa en su lugar (`int` en `List<int>`).
* **Tipo genérico abierto / cerrado:** `List<T>` (sin tipo concreto) frente a `List<int>` (con tipo concreto).
* **Inferencia de tipos:** cuando el compilador deduce el argumento de tipo a partir de los argumentos del método.
* **`default`:** el valor por defecto de un tipo, útil cuando no sabes si `T` es de valor o de referencia.

-----

## El problema

Quieres una "caja" que guarde un valor. Sin genéricos tienes dos caminos, y ambos son malos.

**Camino 1: una clase por tipo.**

```csharp
class CajaInt { public int Valor; }
class CajaString { public string Valor = ""; }
class CajaCliente { public Cliente? Valor; }
// ... y una más por cada tipo nuevo, todas idénticas
```

Duplicas el mismo código una y otra vez.

**Camino 2: usar `object`.**

```csharp
class Caja { public object? Valor; }

var c = new Caja();
c.Valor = 42;                 // boxing
string s = (string)c.Valor;   // compila... y lanza InvalidCastException al ejecutar
```

Funciona para todos los tipos, pero perdiste la verificación del compilador (el error aparece en producción), tienes que hacer casts en todas partes y cada tipo de valor paga boxing.

Antes de .NET 2.0, las colecciones eran así (`ArrayList`, `Hashtable`). Los genéricos dan lo mejor de los dos caminos: **un solo código y seguridad de tipos**.

-----

## Cómo funciona

### Clases genéricas

Se agrega el parámetro de tipo entre `< >` después del nombre:

```csharp
class Caja<T>
{
    public T Valor { get; set; }

    public Caja(T valor) => Valor = valor;

    public bool EstaVacia() => Valor is null;

    public override string ToString() => $"Caja<{typeof(T).Name}>({Valor})";
}
```

Dentro de la clase, `T` se usa como cualquier tipo: en campos, propiedades, parámetros y retornos. Al usar la clase, eliges el tipo:

```csharp
var numeros = new Caja<int>(42);
var texto = new Caja<string>("hola");

int n = numeros.Valor;          // sin cast: el compilador sabe que es int
string s = texto.Valor;

numeros.Valor = "hola";         // error CS0029: no se puede convertir 'string' a 'int'

Console.WriteLine(numeros);     // Caja<Int32>(42)
```

`Caja<int>` y `Caja<string>` son **tipos distintos**, cada uno con sus propias reglas. El error aparece **al compilar**, no al ejecutar.

### Varios parámetros de tipo

```csharp
class Par<TPrimero, TSegundo>
{
    public TPrimero Primero { get; }
    public TSegundo Segundo { get; }

    public Par(TPrimero primero, TSegundo segundo) => (Primero, Segundo) = (primero, segundo);
}

var p = new Par<string, int>("edad", 30);
```

Convenciones de nombres:

* Un solo parámetro: `T`.
* Varios, o cuando el nombre aclara: `T` + descripción (`TKey`, `TValue`, `TResult`, `TEntidad`).

### Genéricos que ya usaste

| Tipo | Parámetros |
| --- | --- |
| `List<T>` | El tipo de los elementos |
| `Dictionary<TKey, TValue>` | El tipo de la clave y el del valor |
| `Nullable<T>` (`int?`) | El tipo de valor que acepta null |
| `Func<T, TResult>`, `Action<T>` | Los tipos de parámetros y de retorno |
| `IEnumerable<T>` | El tipo de los elementos recorridos |
| `Task<TResult>` | El tipo del resultado asíncrono |

### Métodos genéricos

Un método puede tener sus propios parámetros de tipo, **independientes** de los de la clase (y puede estar en una clase no genérica):

```csharp
static class Utilidades
{
    public static void Intercambiar<T>(ref T a, ref T b)
    {
        T temp = a;
        a = b;
        b = temp;
    }

    public static string Describir<T>(T item) => $"{typeof(T).Name}: {item}";
}
```

```csharp
int x = 1, y = 2;
Utilidades.Intercambiar<int>(ref x, ref y);   // tipo explícito
Utilidades.Intercambiar(ref x, ref y);        // tipo INFERIDO de los argumentos (lo habitual)

Console.WriteLine(Utilidades.Describir("hola"));   // String: hola
Console.WriteLine(Utilidades.Describir(3.5));      // Double: 3.5
```

La **inferencia** funciona a partir de los argumentos. Si no puede deducir el tipo, debes indicarlo:

```csharp
Utilidades.Describir(null);            // error CS0411: no se pueden inferir los argumentos de tipo
Utilidades.Describir<string?>(null);   // ✅
```

### `default`: el valor "vacío" de un `T` desconocido

Dentro de un genérico no sabes si `T` es un tipo de valor (`int`) o de referencia (`string`). Por eso `null` no siempre es válido:

```csharp
static T PrimeroODefecto<T>(T[] items)
{
    if (items.Length == 0)
        return null;          // error CS0403: no se puede convertir null al parámetro de tipo 'T'
    return items[0];
}
```

`default` devuelve el valor por defecto del tipo, sea cual sea: `0` para `int`, `null` para `string`, `false` para `bool`:

```csharp
static T? PrimeroODefecto<T>(T[] items) => items.Length == 0 ? default : items[0];

Console.WriteLine(PrimeroODefecto(new int[0]));       // 0
Console.WriteLine(PrimeroODefecto(new string[0]));    // línea vacía: devolvió null
```

`default(T)` es la forma larga; desde C# 7.1 basta con `default` cuando el tipo se deduce.

### Interfaces genéricas

```csharp
interface IRepositorio<T>
{
    void Agregar(T entidad);
    T? ObtenerPorId(int id);
    IReadOnlyList<T> ObtenerTodos();
}
```

Una clase genérica puede implementarla pasando su propio `T`:

```csharp
class RepositorioEnMemoria<T> : IRepositorio<T>
{
    private readonly Dictionary<int, T> _datos = new();
    private int _siguienteId = 1;

    public void Agregar(T entidad) => _datos[_siguienteId++] = entidad;
    public T? ObtenerPorId(int id) => _datos.TryGetValue(id, out var e) ? e : default;
    public IReadOnlyList<T> ObtenerTodos() => _datos.Values.ToList();
}
```

O una clase no genérica puede implementarla con un tipo concreto:

```csharp
class RepositorioClientes : IRepositorio<Cliente> { /* ... */ }
```

### Por qué los genéricos son rápidos

Para tipos de valor, el runtime genera código **especializado**: `List<int>` guarda `int` directamente, sin boxing. Para tipos de referencia, comparte una sola implementación. Por eso `List<int>` es mucho más eficiente que `ArrayList`, que convierte cada número en un objeto del heap.

| | `ArrayList` / `object` | `List<T>` |
| --- | --- | --- |
| Error de tipo | Al **ejecutar** (`InvalidCastException`) | Al **compilar** |
| Casts | En cada lectura | Ninguno |
| Boxing de tipos de valor | Sí, en cada elemento | No |

### Miembros estáticos por tipo cerrado

Cada tipo cerrado tiene sus **propios** campos estáticos:

```csharp
class Contador<T>
{
    public static int Instancias;
    public Contador() => Instancias++;
}

new Contador<int>();
new Contador<int>();
new Contador<string>();

Console.WriteLine(Contador<int>.Instancias);      // 2
Console.WriteLine(Contador<string>.Instancias);   // 1
```

-----

## Ejemplo completo

```csharp
var clientes = new RepositorioEnMemoria<Cliente>();
clientes.Agregar(new Cliente("Ana", "ana@mail.com"));
clientes.Agregar(new Cliente("Luis", "luis@mail.com"));

var productos = new RepositorioEnMemoria<Producto>();
productos.Agregar(new Producto("Teclado", 150m));

Console.WriteLine(clientes.ObtenerPorId(2));
Console.WriteLine(clientes.ObtenerPorId(99)?.ToString() ?? "Cliente no encontrado");
Console.WriteLine($"Productos: {productos.ObtenerTodos().Count}");

var resultado = Resultado<decimal>.Ok(150m);
var error = Resultado<decimal>.Fallo("Precio no disponible");
Mostrar(resultado);
Mostrar(error);

var par = Utilidades.CrearPar("stock", 42);
Console.WriteLine($"{par.Clave} = {par.Valor}");

static void Mostrar<T>(Resultado<T> r) =>
    Console.WriteLine(r.Exito ? $"OK: {r.Valor}" : $"Error: {r.Mensaje}");

record Cliente(string Nombre, string Email);
record Producto(string Nombre, decimal Precio);

interface IRepositorio<T>
{
    void Agregar(T entidad);
    T? ObtenerPorId(int id);
    IReadOnlyList<T> ObtenerTodos();
}

class RepositorioEnMemoria<T> : IRepositorio<T>
{
    private readonly Dictionary<int, T> _datos = new();
    private int _siguienteId = 1;

    public void Agregar(T entidad) => _datos[_siguienteId++] = entidad;
    public T? ObtenerPorId(int id) => _datos.TryGetValue(id, out var e) ? e : default;
    public IReadOnlyList<T> ObtenerTodos() => _datos.Values.ToList();
}

class Resultado<T>
{
    public bool Exito { get; }
    public T? Valor { get; }
    public string? Mensaje { get; }

    private Resultado(bool exito, T? valor, string? mensaje) => (Exito, Valor, Mensaje) = (exito, valor, mensaje);

    public static Resultado<T> Ok(T valor) => new(true, valor, null);
    public static Resultado<T> Fallo(string mensaje) => new(false, default, mensaje);
}

static class Utilidades
{
    public static (TClave Clave, TValor Valor) CrearPar<TClave, TValor>(TClave clave, TValor valor) => (clave, valor);
}
```

Salida:

```text
Cliente { Nombre = Luis, Email = luis@mail.com }
Cliente no encontrado
Productos: 1
OK: 150
Error: Precio no disponible
stock = 42
```

Un solo `RepositorioEnMemoria<T>` sirve para clientes y productos, y un solo `Resultado<T>` para cualquier tipo de dato. En `CrearPar("stock", 42)` el compilador infiere `TClave = string` y `TValor = int`.

-----

## Errores comunes

**1. Usar un valor del tipo equivocado.**
Qué pasa: `new Caja<int>(1).Valor = "A";` da `error CS0029: Cannot implicitly convert type 'string' to 'int'`.
Por qué: `Caja<int>` solo acepta `int`. Es exactamente la protección que buscabas: el error es de **compilación**, no una excepción en ejecución.
Arreglo: usa el tipo correcto o crea una `Caja<string>`.

**2. Devolver `null` desde un `T`.**
Qué pasa: `error CS0403: Cannot convert null to type parameter 'T' because it could be a non-nullable value type. Consider using 'default(T)' instead.`
Por qué: `T` podría ser `int`.
Arreglo: `return default;` o restringe `T` a tipos de referencia (`where T : class`, próxima lección).

**3. No poder inferir el tipo.**
Qué pasa: `error CS0411: The type arguments for method 'Describir<T>(T)' cannot be inferred from the usage. Try specifying the type arguments explicitly.`
Por qué: el compilador no tiene de dónde deducir `T` (por ejemplo, con `null` o si `T` solo aparece en el retorno).
Arreglo: indícalo: `Describir<string?>(null)`.

**4. Usar operadores o miembros que `T` no garantiza.**
Qué pasa: `a + b` con `T a, T b` da `error CS0019: Operator '+' cannot be applied to operands of type 'T' and 'T'`; `item.Nombre` da `error CS1061`.
Por qué: para el compilador, `T` puede ser cualquier tipo, y no todos tienen `+` o `Nombre`.
Arreglo: restricciones (`where T : IEntidad`, `where T : INumber<T>`), en la [próxima lección](05-Restricciones%20y%20varianza.md).

**5. Llamar a un método estático genérico a través de un objeto.**
Qué pasa: `error CS0176: Member cannot be accessed with an instance reference`.
Por qué: los miembros estáticos se usan con el nombre del tipo.
Arreglo: `Utilidades.Describir(...)`.

-----

## Según la versión de C#

* **C# 2 / .NET 2.0:** genéricos y colecciones genéricas (`List<T>`, `Dictionary<TKey, TValue>`), que reemplazaron a `ArrayList` y `Hashtable`.
* **C# 4:** varianza en interfaces y delegados genéricos (`in` / `out`).
* **C# 7.1:** literal `default` sin tipo.
* **C# 9:** `T?` sin restricciones (en tipos de referencia significa "puede ser null"; en tipos de valor, nada cambia).
* **C# 11:** atributos genéricos (`[MiAtributo<T>]`) y *generic math* con miembros `static abstract`.

-----

## Cuándo sí y cuándo no

**Usa genéricos cuando:**

* La misma lógica sirve para varios tipos: colecciones, repositorios, cachés, resultados (`Resultado<T>`), utilidades.
* Te encuentras escribiendo `object` y casts para "aceptar cualquier cosa".

**No uses genéricos cuando:**

* Solo hay un tipo concreto: un `GestorDeFacturas<T>` que solo se usa con `Factura` agrega complejidad sin beneficio.
* La lógica realmente depende del tipo concreto: probablemente necesitas polimorfismo o sobrecargas.

-----

## Resumen en 5 líneas

1. `class Caja<T>` define un tipo con un parámetro de tipo; `Caja<int>` y `Caja<string>` son tipos distintos.
2. Los errores de tipo aparecen al compilar, sin casts ni boxing (a diferencia de `object` y `ArrayList`).
3. Los métodos genéricos (`M<T>(T x)`) tienen sus propios parámetros de tipo, normalmente inferidos.
4. `default` da el valor por defecto de `T`, sea de valor o de referencia; `null` no siempre es válido.
5. Las interfaces genéricas (`IRepositorio<T>`) definen contratos reutilizables para cualquier tipo.

-----

## Para profundizar

<details>
<summary>Reflexión sobre tipos genéricos</summary>

En ejecución se puede inspeccionar y construir tipos genéricos:

```csharp
Type abierto = typeof(Caja<>);                 // definición genérica abierta
Type cerrado = typeof(Caja<int>);              // tipo cerrado

Console.WriteLine(abierto.IsGenericTypeDefinition);   // True
Console.WriteLine(cerrado.IsGenericType);             // True
Console.WriteLine(cerrado.GetGenericArguments()[0]);  // System.Int32
Console.WriteLine(cerrado.GetGenericTypeDefinition() == abierto);   // True

// Construir Caja<string> en ejecución
Type construido = abierto.MakeGenericType(typeof(string));
object? instancia = Activator.CreateInstance(construido, "hola");
Console.WriteLine(instancia);                         // Caja<String>(hola)
```

Lo usan los contenedores de inyección de dependencias (para registrar `IRepositorio<>` → `Repositorio<>` una sola vez) y los frameworks de serialización.

</details>

<details>
<summary>Generic math (C# 11)</summary>

Con las interfaces `INumber<T>`, `IAdditionOperators<...>` y miembros `static abstract`, se pueden escribir algoritmos numéricos genéricos:

```csharp
using System.Numerics;

static T Sumar<T>(IEnumerable<T> valores) where T : INumber<T>
{
    T total = T.Zero;
    foreach (T v in valores) total += v;
    return total;
}

Console.WriteLine(Sumar(new[] { 1, 2, 3 }));          // 6
Console.WriteLine(Sumar(new[] { 1.5m, 2.5m }));       // 4.0
```

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los genéricos permiten escribir clases y métodos que funcionan con cualquier tipo, indicando un parámetro de tipo como `T`. Por ejemplo, `List<T>` sirve para `List<int>` o `List<string>`. Ofrecen seguridad de tipos en compilación y evitan los casts y el boxing que había con `object` o `ArrayList`.

### Respuesta ampliada (semi-senior)

En .NET, los genéricos están reificados: el runtime conoce los argumentos de tipo, a diferencia del *type erasure* de Java. El JIT genera código especializado para cada instanciación con tipos de valor (sin boxing) y comparte una implementación para los de referencia. Cada tipo genérico cerrado tiene sus propios campos estáticos. Los métodos genéricos usan inferencia de tipos a partir de los argumentos. `default` resuelve el valor nulo/cero sin conocer la naturaleza de `T`, y las restricciones (`where`) permiten usar miembros concretos de `T`. Se combinan con varianza (`in`/`out`) en interfaces y delegados, y desde C# 11 con generic math.

### Preguntas frecuentes de seguimiento

**1. ¿Qué ventaja tiene `List<int>` sobre `ArrayList`?**
Seguridad de tipos al compilar, sin casts y sin boxing de cada elemento.

**2. ¿Qué hace `default(T)`?**
Devuelve el valor por defecto del tipo: `0` para numéricos, `false` para `bool`, `null` para referencias.

**3. ¿Los genéricos de C# son como los de Java?**
No. Java borra los tipos al compilar (*type erasure*); .NET los conserva en ejecución, lo que permite reflexión sobre `T`, `new T()` y especialización para tipos de valor.

-----

## Práctica

**Ejercicio 1.** Crea una clase genérica `Pila<T>` con un array interno de capacidad fija, y los métodos `Apilar(T item)`, `Desapilar()` (que devuelve el último) y la propiedad `Cantidad`. Úsala con `int` y con `string`.

<details>
<summary>Solución</summary>

```csharp
var numeros = new Pila<int>(3);
numeros.Apilar(1);
numeros.Apilar(2);
Console.WriteLine(numeros.Desapilar());   // 2
Console.WriteLine(numeros.Cantidad);      // 1

var palabras = new Pila<string>(2);
palabras.Apilar("hola");
Console.WriteLine(palabras.Desapilar());  // hola

class Pila<T>
{
    private readonly T[] _items;
    public int Cantidad { get; private set; }

    public Pila(int capacidad) => _items = new T[capacidad];

    public void Apilar(T item)
    {
        if (Cantidad == _items.Length) throw new InvalidOperationException("La pila está llena.");
        _items[Cantidad++] = item;
    }

    public T Desapilar()
    {
        if (Cantidad == 0) throw new InvalidOperationException("La pila está vacía.");
        return _items[--Cantidad];
    }
}
```

.NET ya trae `Stack<T>`, que se ve en el módulo de colecciones. Implementarla una vez ayuda a entender cómo funciona por dentro.

</details>

**Ejercicio 2.** Escribe un método genérico `Ultimo<T>(IList<T> lista)` que devuelva el último elemento o `default` si la lista está vacía. Pruébalo con una lista de `int` vacía y una de `string` con datos.

<details>
<summary>Solución</summary>

```csharp
Console.WriteLine(Ultimo(new List<int>()));                  // 0
Console.WriteLine(Ultimo(new List<string> { "a", "b" }));    // b

static T? Ultimo<T>(IList<T> lista) => lista.Count == 0 ? default : lista[lista.Count - 1];
```

Fíjate que con `int` "vacío" y "el último es 0" son indistinguibles. Por eso, en APIs reales, este patrón suele ir acompañado de un `TryUltimo(out T valor)` o de un tipo como `Resultado<T>`.

</details>

-----

## Siguiente lección

[Restricciones y varianza](05-Restricciones%20y%20varianza.md)
