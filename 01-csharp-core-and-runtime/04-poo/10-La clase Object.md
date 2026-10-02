# La clase Object

## En una frase

`object` (alias de `System.Object`) es la raíz de **todos** los tipos de C#: cualquier valor puede guardarse en una variable `object` y todos heredan sus métodos `ToString()`, `Equals()`, `GetHashCode()` y `GetType()`, los tres primeros sobrescribibles.

-----

## Antes de empezar

Conviene que ya sepas:

* Herencia, `virtual`/`override` y casting, de [Polimorfismo y casting](09-Polimorfismo%20y%20casting.md).
* La diferencia entre tipos de valor y de referencia, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`object` / `System.Object`:** la clase base de todos los tipos.
* **`ToString()`:** devuelve una representación en texto del objeto.
* **`Equals()`:** indica si dos objetos se consideran iguales.
* **`GetHashCode()`:** devuelve un número usado por los diccionarios y conjuntos para ubicar objetos rápidamente.
* **`GetType()`:** devuelve el tipo real del objeto en tiempo de ejecución.
* **Igualdad por referencia:** dos variables son iguales si apuntan al mismo objeto.
* **Igualdad por valor:** dos objetos son iguales si tienen los mismos datos.
* **Boxing:** envolver un tipo de valor en un objeto del heap para tratarlo como `object`.

-----

## El problema

Desde la primera lección usas `Console.WriteLine()` con `int`, `bool`, `string`, `DateTime`... y hasta con tus propias clases. ¿Cómo puede un solo método aceptar **cualquier** tipo, incluso uno que escribiste ayer?

Y cuando imprimes un objeto tuyo:

```csharp
var p = new Producto("Teclado", 150m);
Console.WriteLine(p);
```

aparece algo inútil como `Producto`. Y si creas dos productos con los mismos datos, `p1.Equals(p2)` da `false`. Todo eso viene de la misma clase: `object`.

-----

## Cómo funciona

### Todo hereda de `object`

Cuando escribes:

```csharp
class Libro { }
```

C# entiende:

```csharp
class Libro : object { }
```

Y si la clase ya tiene una base (`class Libro : Publicacion`), `object` está más arriba en la cadena. Siempre está en la cima de la jerarquía, también para los tipos de valor:

```text
                      object
         ┌──────────────┼───────────────┐
      string         ValueType        tus clases
                  ┌─────┼──────┐          │
                 int   bool  struct     Libro
```

Por eso cualquier valor se puede asignar a una variable `object` (un upcasting):

```csharp
object o1 = new Libro();
object o2 = new Random();
object o3 = 21;              // int → object (boxing)
object o4 = false;
object o5 = "¡Hola!";
```

### ¿Por qué no usar `object` para todo?

Porque **el tipo de la referencia limita lo que puedes hacer** (ver tipo estático en [Polimorfismo y casting](09-Polimorfismo%20y%20casting.md)):

```csharp
object o = "hola";
Console.WriteLine(o.Length);   // error CS1061: 'object' no contiene una definición para 'Length'
```

Perderías toda la funcionalidad específica del tipo y toda la verificación del compilador. `object` es útil cuando de verdad no sabes el tipo (por ejemplo, datos heterogéneos), pero hoy los **genéricos** cubren la mayoría de esos casos con seguridad de tipos.

### Los miembros de `object`

| Miembro | Qué hace | ¿Se puede sobrescribir? |
| --- | --- | --- |
| `ToString()` | Representación en texto. Por defecto, el nombre completo del tipo. | Sí (`virtual`) |
| `Equals(object?)` | Igualdad. Por defecto: por referencia (clases) o por valor de los campos (structs). | Sí (`virtual`) |
| `GetHashCode()` | Código hash para diccionarios y conjuntos. | Sí (`virtual`) |
| `GetType()` | El tipo real del objeto en ejecución. | No |
| `object.ReferenceEquals(a, b)` | ¿Son la misma instancia? (estático) | No |
| `MemberwiseClone()` | Copia superficial del objeto (`protected`). | No |
| `Finalize()` | Finalizador que llama el GC (se escribe como `~Clase()`). | Sí (con sintaxis especial) |

```csharp
object o1 = new object();

Type t = o1.GetType();              // System.Object
string s = o1.ToString()!;          // "System.Object"

object o2 = o1;
bool iguales = o1.Equals(o2);       // True: es la misma instancia
bool otra = o1.Equals(new object()); // False: otra instancia
```

### Sobrescribir `ToString()`

La versión por defecto devuelve el nombre del tipo, lo que rara vez sirve. Sobrescríbela para que tus objetos se muestren bien:

```csharp
var d = new Diario(7);
Console.WriteLine(d);              // Diario en la página 7
Console.WriteLine($"Estado: {d}"); // Estado: Diario en la página 7

class Diario
{
    public int PaginaActual { get; }
    public Diario(int pagina) => PaginaActual = pagina;

    public override string ToString() => $"Diario en la página {PaginaActual}";
}
```

Fíjate que no llamamos a `ToString()` explícitamente: `Console.WriteLine(object)` y la interpolación lo llaman por ti. Por eso `Console.WriteLine` acepta cualquier tipo. Estas dos líneas producen lo mismo:

```csharp
Console.WriteLine(d);
Console.WriteLine(d.ToString());
```

Con una diferencia importante: si `d` es `null`, la primera imprime una línea vacía y la segunda lanza `NullReferenceException`.

`ToString()` también es lo que ves en el depurador y en los logs. Un buen `ToString()` ahorra mucho tiempo de depuración.

### `Equals()`: igualdad por referencia o por valor

Para las clases, `Equals` por defecto compara **referencias**:

```csharp
var a = new Punto(1, 2);
var b = new Punto(1, 2);
Console.WriteLine(a.Equals(b));   // False: son dos objetos distintos
Console.WriteLine(a == b);        // False: == también compara referencias en clases
```

Si para tu tipo "mismos datos" significa "mismo objeto conceptual", sobrescribe `Equals` **y** `GetHashCode` juntos:

```csharp
class Punto
{
    public int X { get; }
    public int Y { get; }
    public Punto(int x, int y) => (X, Y) = (x, y);

    public override bool Equals(object? obj) =>
        obj is Punto otro && X == otro.X && Y == otro.Y;

    public override int GetHashCode() => HashCode.Combine(X, Y);

    public override string ToString() => $"({X}, {Y})";
}
```

```csharp
var a = new Punto(1, 2);
var b = new Punto(1, 2);
Console.WriteLine(a.Equals(b));               // True: mismos datos
Console.WriteLine(object.ReferenceEquals(a, b)); // False: siguen siendo dos instancias
```

### Por qué `GetHashCode` va siempre con `Equals`

Los diccionarios (`Dictionary<K, V>`) y los conjuntos (`HashSet<T>`) primero usan `GetHashCode()` para encontrar el "casillero" de un objeto, y después `Equals()` para confirmar. La regla es:

> **Si dos objetos son iguales según `Equals`, deben tener el mismo `GetHashCode`.**

Si sobrescribes `Equals` y no `GetHashCode`, el compilador te avisa (CS0659) y los diccionarios fallan en silencio: dos puntos "iguales" pueden caer en casilleros distintos y el conjunto tendrá duplicados.

`HashCode.Combine(...)` genera un buen hash a partir de los mismos campos que usas en `Equals`.

### `==` frente a `Equals`

* `Equals` es un método virtual: se decide por el tipo **real** del objeto.
* `==` es un operador estático: se decide por el tipo **declarado** de las variables, y solo cambia si el tipo lo sobrecarga.

```csharp
string s1 = "hola";
string s2 = new string("hola".ToCharArray());

Console.WriteLine(s1 == s2);            // True: string sobrecarga == para comparar contenido

object o1 = s1, o2 = s2;
Console.WriteLine(o1 == o2);            // False: con tipo object, == compara referencias
Console.WriteLine(o1.Equals(o2));       // True: Equals es virtual → usa la versión de string
```

Por eso, si sobrescribes `Equals` en tu clase, conviene también sobrecargar `==` y `!=` para que se comporten igual (o usar un `record`, que lo hace todo por ti).

### Boxing: los tipos de valor como `object`

Cuando asignas un tipo de valor a `object`, el runtime crea una "caja" en el heap con una **copia** del valor:

```csharp
int n = 5;
object caja = n;        // boxing: nuevo objeto en el heap
n = 99;
Console.WriteLine(caja);    // 5: la caja tiene su propia copia

int m = (int)caja;      // unboxing: hay que hacer un cast al tipo exacto
long l = (long)caja;    // InvalidCastException: la caja contiene un int, no un long
```

El boxing tiene un costo (memoria y trabajo del GC). Los genéricos (`List<int>` en lugar de una lista de `object`) existen en buena parte para evitarlo.

-----

## Ejemplo completo

```csharp
var p1 = new Producto("A-001", "Teclado", 150m);
var p2 = new Producto("A-001", "Teclado mecánico", 180m);   // mismo código, otro nombre y precio
var p3 = new Producto("B-002", "Mouse", 60m);

Console.WriteLine(p1);                          // usa ToString()
Console.WriteLine($"p1.Equals(p2): {p1.Equals(p2)}");
Console.WriteLine($"p1 == p2: {p1 == p2}");
Console.WriteLine($"Misma instancia: {ReferenceEquals(p1, p2)}");

var catalogo = new HashSet<Producto> { p1, p2, p3 };   // usa GetHashCode + Equals
Console.WriteLine($"Productos únicos en el catálogo: {catalogo.Count}");

object[] cosas = { p1, 42, "texto", DateTime.Today, null! };
foreach (object? cosa in cosas)
{
    string tipo = cosa?.GetType().Name ?? "null";
    Console.WriteLine($"{tipo,-10} → {cosa}");
}

class Producto
{
    public string Codigo { get; }
    public string Nombre { get; }
    public decimal Precio { get; }

    public Producto(string codigo, string nombre, decimal precio)
    {
        Codigo = codigo;
        Nombre = nombre;
        Precio = precio;
    }

    // Dos productos son "el mismo" si tienen el mismo código (su identidad de negocio)
    public override bool Equals(object? obj) => obj is Producto otro && Codigo == otro.Codigo;
    public override int GetHashCode() => Codigo.GetHashCode();
    public override string ToString() => $"[{Codigo}] {Nombre} ({Precio:N2})";

    public static bool operator ==(Producto? a, Producto? b) => Equals(a, b);
    public static bool operator !=(Producto? a, Producto? b) => !Equals(a, b);
}
```

Salida (la fecha varía):

```text
[A-001] Teclado (150.00)
p1.Equals(p2): True
p1 == p2: True
Misma instancia: False
Productos únicos en el catálogo: 2
Producto   → [A-001] Teclado (150.00)
Int32      → 42
String     → texto
DateTime   → 10/2/2026 12:00:00 AM
null       → 
```

El `HashSet` descartó `p2` porque, según `Equals` y `GetHashCode`, es el mismo producto que `p1`. El array `object[]` mezcla tipos distintos y cada uno se imprime con **su** `ToString()`. `Equals(a, b)` dentro de los operadores es el método estático `object.Equals`, que maneja los `null` y luego llama al `Equals` virtual.

-----

## Errores comunes

**1. Sobrescribir `Equals` sin `GetHashCode`.**
Qué pasa: `warning CS0659: 'Punto' overrides Object.Equals(object o) but does not override Object.GetHashCode()`, y los diccionarios o conjuntos se comportan mal (duplicados, búsquedas que no encuentran nada).
Por qué: objetos iguales deben tener el mismo hash.
Arreglo: sobrescribe los dos, con los mismos campos: `HashCode.Combine(X, Y)`.

**2. Esperar que `==` compare contenido en tus clases.**
Qué pasa: `p1 == p2` da `False` aunque `Equals` dé `True`.
Por qué: `==` en clases compara referencias salvo que lo sobrecargues.
Arreglo: sobrecarga `==` y `!=`, o usa un `record`.

**3. Llamar a `ToString()` sobre `null`.**
Qué pasa: `NullReferenceException`.
Por qué: no hay objeto sobre el cual llamar al método.
Arreglo: usa `obj?.ToString()` o deja que la interpolación o `Console.WriteLine` lo manejen.

**4. Unboxing a un tipo distinto del original.**
Qué pasa: `System.InvalidCastException: Unable to cast object of type 'System.Int32' to type 'System.Int64'.`
Por qué: el unboxing exige el tipo exacto que se guardó en la caja.
Arreglo: `(long)(int)caja`, o mejor `Convert.ToInt64(caja)`.

**5. Usar campos que cambian en `GetHashCode`.**
Qué pasa: un objeto agregado a un `HashSet` "desaparece" después de modificarlo.
Por qué: su hash cambió y ahora el conjunto lo busca en otro casillero.
Arreglo: calcula el hash con datos inmutables (como `Codigo`, de solo lectura).

-----

## Según la versión de C#

* **.NET Core 2.1:** `HashCode.Combine(...)` para generar códigos hash sin fórmulas manuales.
* **C# 8:** `string?` y `object?` en las firmas: `Equals(object? obj)` y `ToString()` que puede devolver `null`.
* **C# 9:** los `record` generan automáticamente `Equals`, `GetHashCode`, `==`, `!=` y un `ToString()` legible a partir de sus propiedades.

```csharp
public record Punto(int X, int Y);

var a = new Punto(1, 2);
var b = new Punto(1, 2);
Console.WriteLine(a == b);   // True
Console.WriteLine(a);        // Punto { X = 1, Y = 2 }
```

Si lo único que necesitas es igualdad por valor y un buen `ToString()`, un `record` es mucho menos código y menos propenso a errores que sobrescribir todo a mano.

-----

## Cuándo sí y cuándo no

**Sobrescribe `ToString()` cuando:**

* Casi siempre en clases de dominio: mejora los logs, el depurador y los mensajes de error.

**Sobrescribe `Equals` y `GetHashCode` cuando:**

* Dos instancias con los mismos datos de identidad deben considerarse el mismo objeto (y especialmente si se usan como clave de diccionario o en conjuntos).

**Usa un `record` cuando:**

* El tipo es principalmente datos y quieres igualdad por valor sin escribirla.

**Usa `object` como tipo cuando:**

* De verdad no conoces el tipo y no hay forma de usar genéricos (APIs antiguas, reflexión, serialización dinámica).

-----

## Resumen en 5 líneas

1. Todos los tipos derivan de `object`, también los de valor (a través de `ValueType`).
2. `object` aporta `ToString()`, `Equals()`, `GetHashCode()` y `GetType()`; los tres primeros son virtuales.
3. `Console.WriteLine` y la interpolación llaman a `ToString()`: sobrescríbelo para mostrar algo útil.
4. Si sobrescribes `Equals`, sobrescribe también `GetHashCode` con los mismos campos.
5. Asignar un tipo de valor a `object` produce boxing; los `record` generan igualdad por valor automáticamente.

-----

## Para profundizar

<details>
<summary>IEquatable&lt;T&gt;: igualdad sin boxing</summary>

`Equals(object?)` recibe un `object`, así que comparar structs con él provoca boxing. Implementar `IEquatable<T>` agrega un `Equals(T)` fuertemente tipado que usan `List<T>.Contains`, `Dictionary` y LINQ:

```csharp
readonly struct Coordenada : IEquatable<Coordenada>
{
    public double Lat { get; }
    public double Lon { get; }
    public Coordenada(double lat, double lon) => (Lat, Lon) = (lat, lon);

    public bool Equals(Coordenada otra) => Lat == otra.Lat && Lon == otra.Lon;
    public override bool Equals(object? obj) => obj is Coordenada c && Equals(c);
    public override int GetHashCode() => HashCode.Combine(Lat, Lon);
}
```

Los records y los `record struct` ya implementan `IEquatable<T>`.

</details>

<details>
<summary>MemberwiseClone y la copia superficial</summary>

`MemberwiseClone()` crea un objeto nuevo copiando cada campo. Para los campos de tipo de referencia copia la **referencia**, no el objeto: la copia y el original comparten las listas internas, por ejemplo. Es `protected`, así que se expone con un método propio:

```csharp
public Pedido Clonar() => (Pedido)MemberwiseClone();
```

Para una copia profunda hay que clonar también los objetos internos. Con records, `with` crea copias superficiales modificando algunas propiedades: `var p2 = p1 with { Precio = 10 };`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`System.Object` es la clase base de todos los tipos en .NET, así que cualquier valor se puede guardar en una variable `object`. Todos heredan `ToString()`, `Equals()`, `GetHashCode()` y `GetType()`. Normalmente se sobrescribe `ToString()` para mostrar el objeto de forma legible, y si se sobrescribe `Equals()` también hay que sobrescribir `GetHashCode()`.

### Respuesta ampliada (semi-senior)

`object` es la raíz del sistema de tipos unificado: los tipos de referencia derivan directamente o en cadena, y los de valor a través de `System.ValueType`, que sobrescribe `Equals` para comparar campos (con reflexión, de forma lenta, salvo que se implemente). Convertir un tipo de valor a `object` produce boxing. `Equals` es virtual (despacho por tipo real) y `==` es un operador estático (resuelto por el tipo declarado), lo que explica las diferencias con `string` tratado como `object`. El contrato exige que la igualdad sea reflexiva, simétrica y transitiva, y que objetos iguales tengan el mismo hash, idealmente calculado sobre datos inmutables. Para tipos de datos, los `record` generan `Equals`, `GetHashCode`, `==`, `ToString` e `IEquatable<T>`.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué hay que sobrescribir `GetHashCode` si sobrescribo `Equals`?**
Porque `Dictionary` y `HashSet` usan el hash para ubicar los objetos: dos objetos iguales con hashes distintos rompen esas colecciones.

**2. ¿Diferencia entre `==` y `Equals`?**
`Equals` es virtual y se decide por el tipo real. `==` es estático, se decide por el tipo declarado y, en clases, compara referencias salvo que se sobrecargue.

**3. ¿Qué devuelve `ToString()` por defecto?**
El nombre completo del tipo, con su espacio de nombres (por ejemplo, `MiApp.Producto`).

**4. ¿Los `int` heredan de `object`?**
Sí, a través de `System.ValueType`. Por eso `42.ToString()` funciona, y por eso asignar un `int` a `object` produce boxing.

-----

## Práctica

**Ejercicio 1.** Crea una clase `Fecha` con `Dia`, `Mes` y `Anio` (de solo lectura). Sobrescribe `ToString()` para que muestre `dd/mm/aaaa` con dos dígitos en día y mes, y `Equals`/`GetHashCode` para que dos fechas con los mismos valores sean iguales. Comprueba que un `HashSet<Fecha>` no admite duplicados.

<details>
<summary>Solución</summary>

```csharp
var fechas = new HashSet<Fecha>
{
    new(2, 10, 2026),
    new(2, 10, 2026),
    new(25, 12, 2026)
};

Console.WriteLine(fechas.Count);                 // 2
foreach (var f in fechas) Console.WriteLine(f);  // 02/10/2026, 25/12/2026

class Fecha
{
    public int Dia { get; }
    public int Mes { get; }
    public int Anio { get; }

    public Fecha(int dia, int mes, int anio) => (Dia, Mes, Anio) = (dia, mes, anio);

    public override string ToString() => $"{Dia:00}/{Mes:00}/{Anio}";
    public override bool Equals(object? obj) => obj is Fecha f && Dia == f.Dia && Mes == f.Mes && Anio == f.Anio;
    public override int GetHashCode() => HashCode.Combine(Dia, Mes, Anio);
}
```

</details>

**Ejercicio 2.** Sin ejecutar, ¿qué imprime?

```csharp
object a = 10;
object b = 10;
Console.WriteLine(a == b);
Console.WriteLine(a.Equals(b));
Console.WriteLine(a.GetType().Name);
```

<details>
<summary>Solución</summary>

```text
False
True
Int32
```

Cada asignación hizo boxing en una caja distinta: `==` entre variables `object` compara referencias (`False`). `Equals` es virtual y usa la versión de `Int32`, que compara valores (`True`).

</details>

-----

## Siguiente lección

[Encadenamiento de métodos](11-Encadenamiento%20de%20metodos.md)
