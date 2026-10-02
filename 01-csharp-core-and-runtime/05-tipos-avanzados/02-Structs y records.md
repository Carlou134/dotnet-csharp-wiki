# Structs y records

## En una frase

Un `struct` es un tipo **de valor** que se copia completo al asignarlo (ideal para valores pequeños como una coordenada); un `record` es un tipo pensado para **datos**, con igualdad por valor, `ToString` legible y copias con `with` generados automáticamente, que puede ser de referencia (`record`) o de valor (`record struct`).

-----

## Antes de empezar

Conviene que ya sepas:

* La diferencia entre tipos de valor y de referencia, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).
* Propiedades con `init`, constructores y la clase `object` (`Equals`, `GetHashCode`, `ToString`), de [POO](../04-poo/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`struct`:** tipo de valor definido por el usuario.
* **`readonly struct`:** struct cuyos campos no pueden cambiar después de construirse.
* **`record`:** tipo orientado a datos con igualdad por valor generada por el compilador.
* **Record posicional:** record declarado con parámetros: `record Punto(int X, int Y)`.
* **Expresión `with`:** crea una copia de un record cambiando algunas propiedades.
* **Igualdad por valor:** dos instancias son iguales si sus datos son iguales.
* **Mutación defensiva / copia oculta:** copia que el compilador hace de un struct para protegerlo, a veces sin que lo notes.
* **DTO (*Data Transfer Object*):** objeto cuyo único propósito es transportar datos entre capas o sistemas.

-----

## El problema

Ya conoces las clases. Pero para ciertos datos, las clases traen fricción:

1. Quieres comparar dos coordenadas o dos montos de dinero, y `==` entre clases compara **referencias**. Tienes que sobrescribir `Equals`, `GetHashCode`, `==`, `!=` y `ToString` a mano: 20 líneas de código repetitivo, fácil de equivocar (ver [La clase Object](../04-poo/10-La%20clase%20Object.md)).
2. Quieres un dato **inmutable** ("este pedido con otro estado") y terminas escribiendo un constructor de copia.
3. Tienes millones de coordenadas pequeñas, y cada una es un objeto en el heap que el GC tiene que rastrear.

`record` resuelve los puntos 1 y 2. `struct` resuelve el 3.

-----

## Cómo funciona

### `struct`: un tipo de valor propio

```csharp
struct Punto
{
    public int X { get; set; }
    public int Y { get; set; }

    public Punto(int x, int y)
    {
        X = x;
        Y = y;
    }

    public double DistanciaAlOrigen() => Math.Sqrt(X * X + Y * Y);
}
```

Se declara como una clase, pero con `struct`. La diferencia está en cómo se **copia**:

```csharp
var a = new Punto(1, 2);
var b = a;          // se COPIAN los datos
b.X = 100;

Console.WriteLine(a.X);   // 1: a no cambió
Console.WriteLine(b.X);   // 100
```

Características de un struct:

* Es un tipo de valor: asignar, pasar como argumento o devolverlo **copia** todos sus campos.
* Una variable local de tipo struct no es un objeto aparte en el heap (no presiona al GC).
* Nunca es `null` (salvo que uses `Punto?`).
* No admite herencia (no puede heredar de otro struct ni de una clase, ni ser base de nada), pero **sí** implementa interfaces.
* Siempre existe `default(Punto)`, con todos los campos en su valor por defecto, aunque no declares un constructor sin parámetros.

Ya usaste muchos structs: `int`, `double`, `bool`, `DateTime`, `TimeSpan`, `Guid`, `decimal`.

### `readonly struct`: structs inmutables

Un struct mutable es una fuente clásica de bugs, porque es fácil modificar **una copia** creyendo que modificas el original:

```csharp
var puntos = new List<Punto> { new Punto(1, 2) };
puntos[0].X = 5;   // error CS1612: no se puede modificar el valor devuelto, porque no es una variable
```

`puntos[0]` devuelve una **copia**; modificarla no tendría efecto, y el compilador te frena. La recomendación es hacer los structs **inmutables**:

```csharp
readonly struct Coordenada
{
    public double Lat { get; }
    public double Lon { get; }

    public Coordenada(double lat, double lon) => (Lat, Lon) = (lat, lon);

    public Coordenada MoverNorte(double grados) => new(Lat + grados, Lon);   // devuelve una nueva
}
```

`readonly struct` obliga a que todos los campos sean de solo lectura y permite al compilador evitar copias defensivas.

### `record`: datos con igualdad por valor

```csharp
public record Persona(string Nombre, int Edad);
```

Esa sola línea es un **record posicional**. El compilador genera:

* Propiedades `Nombre` y `Edad` con `get` e `init` (se asignan al crear y no cambian después).
* Un constructor con esos parámetros.
* `Equals`, `GetHashCode`, `==` y `!=` **por valor**.
* Un `ToString()` legible.
* Un método `Deconstruct`.
* Soporte para `with`.

```csharp
var a = new Persona("Ana", 30);
var b = new Persona("Ana", 30);

Console.WriteLine(a == b);                    // True: mismos datos
Console.WriteLine(ReferenceEquals(a, b));     // False: dos objetos distintos
Console.WriteLine(a);                         // Persona { Nombre = Ana, Edad = 30 }

a.Edad = 31;                                  // error CS8852: solo se puede asignar en un inicializador

var mayor = a with { Edad = 31 };             // copia con un cambio
Console.WriteLine(mayor);                     // Persona { Nombre = Ana, Edad = 31 }

var (nombre, edad) = mayor;                   // deconstrucción
```

Un `record` (o `record class`) sigue siendo un **tipo de referencia**: vive en el heap y la variable guarda una referencia. Lo que cambia es la **igualdad**, que pasa a ser por valor.

### Records con cuerpo y records nominales

Puedes agregar miembros, validación y propiedades extra:

```csharp
public record Producto(string Codigo, string Nombre, decimal Precio)
{
    public decimal PrecioConIva => Precio * 1.18m;          // propiedad calculada

    public string Codigo { get; init; } = Codigo.ToUpperInvariant();   // reemplaza la propiedad generada
}
```

O declararlo sin parámetros, como una clase (record *nominal*):

```csharp
public record Cliente
{
    public required string Email { get; init; }
    public string? Telefono { get; init; }
}

var c = new Cliente { Email = "ana@mail.com" };
```

### ¿Los records son inmutables?

A medias, y es una confusión muy común:

* Las propiedades **posicionales** son `init`: no se pueden cambiar después de crear el objeto.
* Pero puedes declarar propiedades con `set` en un record, y entonces es mutable.
* Y la inmutabilidad es **superficial**: si una propiedad es una `List<T>`, no puedes reemplazar la lista, pero sí agregarle elementos.

```csharp
public record Carrito(string Cliente, List<string> Items);

var c = new Carrito("Ana", new List<string>());
c.Items.Add("Teclado");      // ✅ compila: la lista sí es mutable
```

Para inmutabilidad completa, usa colecciones inmutables o de solo lectura (`IReadOnlyList<T>`, `ImmutableList<T>`).

### `record struct`: lo mejor de los dos

```csharp
public record struct Color(byte R, byte G, byte B);
public readonly record struct Dinero(decimal Monto, string Moneda);
```

* `record struct`: tipo de **valor** con igualdad, `ToString` y `with` generados. Sus propiedades posicionales tienen `set` (es mutable).
* `readonly record struct`: igual, pero inmutable. Es la opción recomendada para valores pequeños.

### Comparación

| | `class` | `struct` | `record` (class) | `record struct` |
| --- | --- | --- | --- | --- |
| Tipo | Referencia | Valor | Referencia | Valor |
| Al asignar | Comparte el objeto | Copia los datos | Comparte el objeto | Copia los datos |
| `==` por defecto | Referencia | No definido (solo `Equals`, por campos) | Por valor | Por valor |
| `ToString` legible | No | No | Sí | Sí |
| `with` | No | Sí (C# 10) | Sí | Sí |
| Herencia | Sí | No | Sí (entre records) | No |
| Puede ser `null` | Sí | No (salvo `T?`) | Sí | No (salvo `T?`) |
| Pensado para | Entidades con identidad y comportamiento | Valores pequeños | Datos, DTOs, mensajes | Valores pequeños con igualdad |

-----

## Ejemplo completo

```csharp
// record: DTO que llega de una API
var pedido = new PedidoDto(1042, "Ana", Estado: "Pendiente");
var pagado = pedido with { Estado = "Pagado" };

Console.WriteLine(pedido);
Console.WriteLine(pagado);
Console.WriteLine($"¿Mismo pedido con el mismo estado? {pedido == pagado}");

// readonly record struct: valor pequeño con igualdad y operaciones
var precio = new Dinero(120m, "PEN");
var envio = new Dinero(15m, "PEN");
var total = precio + envio;

Console.WriteLine($"Total: {total}");
Console.WriteLine($"¿135 PEN == total? {new Dinero(135m, "PEN") == total}");

// struct mutable vs. copia
var original = new Contador();
var copia = original;
copia.Incrementar();
Console.WriteLine($"original: {original.Valor}, copia: {copia.Valor}");

// los records sirven como claves de diccionario gracias a la igualdad por valor
var stock = new Dictionary<Ubicacion, int>
{
    [new Ubicacion("A", 1)] = 30,
    [new Ubicacion("B", 4)] = 12
};
Console.WriteLine($"Stock en A-1: {stock[new Ubicacion("A", 1)]}");

public record PedidoDto(int Id, string Cliente, string Estado);

public readonly record struct Dinero(decimal Monto, string Moneda)
{
    public static Dinero operator +(Dinero a, Dinero b)
    {
        if (a.Moneda != b.Moneda)
            throw new InvalidOperationException("No se pueden sumar monedas distintas.");
        return new Dinero(a.Monto + b.Monto, a.Moneda);
    }

    public override string ToString() => $"{Monto:N2} {Moneda}";
}

public record Ubicacion(string Pasillo, int Estante);

public struct Contador
{
    public int Valor { get; private set; }
    public void Incrementar() => Valor++;
}
```

Salida:

```text
PedidoDto { Id = 1042, Cliente = Ana, Estado = Pendiente }
PedidoDto { Id = 1042, Cliente = Ana, Estado = Pagado }
¿Mismo pedido con el mismo estado? False
Total: 135.00 PEN
¿135 PEN == total? True
original: 0, copia: 1
Stock en A-1: 30
```

Cada tipo cumple su papel: el record como DTO inmutable, el `readonly record struct` como valor con operadores y el struct mutable para mostrar la semántica de copia.

-----

## Errores comunes

**1. Modificar un struct a través de una copia.**
Qué pasa: `lista[0].X = 5;` da `error CS1612: Cannot modify the return value of 'List<Punto>.this[int]' because it is not a variable`.
Por qué: el indexador devuelve una copia del struct.
Arreglo: haz el struct inmutable y reemplaza el elemento: `lista[0] = lista[0] with { X = 5 };`.

**2. Asignar una propiedad posicional de un record.**
Qué pasa: `error CS8852: Init-only property or indexer 'Persona.Edad' can only be assigned in an object initializer...`.
Por qué: las propiedades posicionales de un `record` (class) son `init`.
Arreglo: crea una copia con `with { Edad = 31 }`.

**3. Creer que un record es un tipo de valor.**
Qué pasa: compartes un record entre dos variables y esperas que se copie.
Por qué: `record` es `record class`, un tipo de referencia; solo su igualdad es por valor.
Arreglo: si necesitas semántica de valor, usa `record struct`.

**4. Heredar en un struct.**
Qué pasa: `struct A : B` (con `B` una clase o struct) da `error CS0527: Type 'B' in interface list is not an interface`.
Por qué: los structs no admiten herencia, solo interfaces.
Arreglo: usa composición o una clase.

**5. Structs grandes.**
Qué pasa: no hay error, pero el rendimiento empeora: cada asignación o llamada copia muchos bytes.
Por qué: la semántica de valor copia el struct completo.
Arreglo: la guía de .NET recomienda structs de hasta unos 16 bytes. Si es más grande o mutable, usa una clase.

**6. Usar un record como entidad de Entity Framework.**
Qué pasa: comportamientos raros con el seguimiento de cambios, porque dos entidades distintas con los mismos datos se consideran "iguales".
Por qué: EF identifica las entidades por identidad, no por valor.
Arreglo: usa clases para entidades y records para DTOs y objetos de valor.

-----

## Según la versión de C#

* **C# 7.2:** `readonly struct` y `ref struct`.
* **C# 9:** `record` (class), `init` y `with` para records.
* **C# 10:** `record struct`, `readonly record struct`, `with` para cualquier struct y constructores sin parámetros explícitos en structs.
* **C# 11:** `required` y campos de structs que se inicializan automáticamente a `default`.
* **C# 12:** constructores primarios para clases y structs comunes (no solo records).

Si ves en código antiguo una clase con 30 líneas de `Equals`/`GetHashCode`/`ToString`, probablemente hoy sería un record de una línea.

-----

## Cuándo sí y cuándo no

**Usa `record` cuando:**

* El tipo es principalmente **datos**: DTOs de una API, mensajes entre servicios, comandos, resultados de consultas, objetos de valor del dominio.
* Quieres igualdad por valor y copias inmutables con `with`.

**Usa `readonly struct` o `readonly record struct` cuando:**

* El valor es **pequeño** (unos pocos campos), inmutable y se crea en grandes cantidades (coordenadas, colores, dinero).

**Usa `class` cuando:**

* El objeto tiene **identidad** (dos clientes con el mismo nombre siguen siendo dos clientes), estado que cambia y comportamiento: entidades, servicios.

-----

## Resumen en 5 líneas

1. `struct` = tipo de valor propio: se copia al asignar, no admite herencia y no es `null`.
2. Prefiere structs pequeños e inmutables (`readonly struct`) para evitar modificar copias sin querer.
3. `record Persona(string Nombre, int Edad)` genera propiedades `init`, igualdad por valor, `ToString`, `Deconstruct` y `with`.
4. `record` es de referencia; `record struct` es de valor; `readonly record struct` es la versión inmutable.
5. La inmutabilidad de un record es superficial: una lista dentro de un record sigue siendo modificable.

-----

## Para profundizar

<details>
<summary>Herencia de records y EqualityContract</summary>

Los records (class) pueden heredar de otros records:

```csharp
public record Animal(string Nombre);
public record Perro(string Nombre, string Raza) : Animal(Nombre);

Animal a = new Perro("Firulais", "Labrador");
Animal b = new Animal("Firulais");
Console.WriteLine(a == b);   // False
```

Aunque comparten `Nombre`, no son iguales: el compilador genera una propiedad oculta `EqualityContract` con el tipo real, y la igualdad exige que coincida. Así, un `Perro` nunca es igual a un `Animal` genérico.

</details>

<details>
<summary>ref struct y Span&lt;T&gt;</summary>

Un `ref struct` es un struct que **solo puede vivir en el stack**: no se puede guardar en un campo de una clase, ni hacer boxing, ni capturar en una lambda. `Span<T>` y `ReadOnlySpan<T>` son `ref struct`, y por eso pueden apuntar a memoria del stack o a una porción de un array sin copiarla. Se usan en código de alto rendimiento (parsers, serializadores).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un `struct` es un tipo de valor: se copia al asignarlo y se usa para datos pequeños como coordenadas. Un `record` es un tipo pensado para datos, que genera automáticamente igualdad por valor, `ToString` y la expresión `with` para crear copias modificadas. `record` es de referencia y `record struct` es de valor.

### Respuesta ampliada (semi-senior)

Los structs tienen semántica de copia, no admiten herencia y siempre tienen un `default`; se recomiendan pequeños (≤16 bytes) e inmutables (`readonly struct`) para evitar copias defensivas y bugs por modificar copias, y para no pagar boxing deben implementar `IEquatable<T>`. Los records generan `Equals`/`GetHashCode` por valor (incluido un `EqualityContract` para respetar la herencia), `ToString`, `Deconstruct`, un constructor de copia y soporte para `with`. Las propiedades posicionales de `record class` son `init` y las de `record struct` son mutables, salvo con `readonly`. La inmutabilidad es superficial. Se usan para DTOs, mensajes y objetos de valor; para entidades con identidad (por ejemplo, en EF Core) siguen siendo preferibles las clases.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `record` y `class`?**
Ambos son de referencia, pero el `record` tiene igualdad por valor, `ToString` legible y `with` generados. Una clase compara por referencia salvo que sobrescribas `Equals`.

**2. ¿Diferencia entre `struct` y `class`?**
`struct` es de valor (se copia, no admite herencia, no es `null`); `class` es de referencia.

**3. ¿Los records son inmutables?**
Las propiedades posicionales de un `record` son `init`, pero se pueden declarar propiedades con `set`, y las colecciones internas siguen siendo mutables. Es inmutabilidad superficial y opcional.

**4. ¿Cuándo usarías un struct?**
Para valores pequeños, inmutables y muy numerosos, donde evitar asignaciones en el heap mejora el rendimiento.

-----

## Práctica

**Ejercicio 1.** Reemplaza esta clase por un record de una línea y comprueba que dos instancias con los mismos datos son iguales y que puedes crear una copia cambiando el precio.

```csharp
class Libro
{
    public string Isbn { get; }
    public string Titulo { get; }
    public decimal Precio { get; }
    public Libro(string isbn, string titulo, decimal precio) { Isbn = isbn; Titulo = titulo; Precio = precio; }
    public override bool Equals(object? obj) => obj is Libro l && l.Isbn == Isbn && l.Titulo == Titulo && l.Precio == Precio;
    public override int GetHashCode() => HashCode.Combine(Isbn, Titulo, Precio);
    public override string ToString() => $"Libro {{ Isbn = {Isbn}, Titulo = {Titulo}, Precio = {Precio} }}";
}
```

<details>
<summary>Solución</summary>

```csharp
var a = new Libro("978-1", "Clean Code", 45m);
var b = new Libro("978-1", "Clean Code", 45m);
var oferta = a with { Precio = 30m };

Console.WriteLine(a == b);    // True
Console.WriteLine(oferta);    // Libro { Isbn = 978-1, Titulo = Clean Code, Precio = 30 }

public record Libro(string Isbn, string Titulo, decimal Precio);
```

</details>

**Ejercicio 2.** ¿Qué imprime este código? Explica por qué.

```csharp
var s1 = new PuntoS { X = 1 };
var s2 = s1;
s2.X = 9;

var r1 = new PuntoR { X = 1 };
var r2 = r1;
r2.X = 9;

Console.WriteLine($"{s1.X} {r1.X}");

struct PuntoS { public int X; }
record PuntoR { public int X { get; set; } }
```

<details>
<summary>Solución</summary>

`1 9`. `PuntoS` es un struct: `s2` es una copia. `PuntoR` es un record (class), es decir, un tipo de referencia: `r1` y `r2` apuntan al mismo objeto, y como `X` tiene `set`, el cambio se ve en los dos.

</details>

-----

## Siguiente lección

[Tipos que aceptan null](03-Tipos%20que%20aceptan%20null.md)
