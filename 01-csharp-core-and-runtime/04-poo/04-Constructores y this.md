# Constructores y this

## En una frase

Un **constructor** es un método especial, con el mismo nombre que la clase y sin tipo de retorno, que se ejecuta automáticamente al hacer `new` para dejar el objeto **listo y válido desde el primer momento**; `this` se refiere al objeto actual y `: this(...)` encadena un constructor con otro.

-----

## Antes de empezar

Conviene que ya sepas:

* Declarar propiedades, incluidas las de solo lectura, de [Propiedades](03-Propiedades.md).
* Qué es la sobrecarga y qué son los parámetros opcionales, de [Parámetros opcionales, nombrados y sobrecarga](../03-metodos/02-Parametros%20opcionales%20nombrados%20y%20sobrecarga.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Constructor:** método especial que inicializa un objeto al crearlo.
* **Constructor sin parámetros (por defecto):** el que el compilador genera si no declaras ninguno.
* **`this`:** referencia al objeto sobre el que se está ejecutando el código.
* **Encadenamiento de constructores:** un constructor que llama a otro con `: this(...)`.
* **Constructor primario:** parámetros declarados junto al nombre de la clase (C# 12).
* **Inicializador de campo:** el valor asignado en la declaración de un campo o propiedad.

-----

## El problema

Hasta ahora creabas objetos así:

```csharp
var b = new Bosque();
b.Nombre = "Amazonas";
b.Area = 400;
```

Entre la primera y la última línea, el objeto existe **a medio construir**: un bosque sin nombre. Si olvidas una asignación, el objeto queda incompleto para siempre y el error aparece mucho después, lejos de donde se originó.

Lo que quieres es que **sea imposible** crear un `Bosque` sin los datos obligatorios. Que la regla "todo bosque tiene un nombre" la garantice la propia clase, no la memoria de quien la usa.

-----

## Cómo funciona

### Declarar un constructor

```csharp
class Bosque
{
    public string Nombre { get; }
    public int Area { get; private set; }

    public Bosque(string nombre, int area)      // constructor
    {
        Nombre = nombre;
        Area = area;
    }
}
```

* Se llama **igual que la clase**.
* **No tiene tipo de retorno**, ni siquiera `void`.
* Se ejecuta automáticamente con `new`:

```csharp
var b = new Bosque("Amazonas", 400);   // aquí se ejecuta el constructor
Console.WriteLine(b.Nombre);           // Amazonas
```

Ahora un bosque no se puede crear sin nombre ni área: el compilador lo exige.

### Validar en el constructor

El constructor es el lugar ideal para garantizar que el objeto **nazca válido**:

```csharp
public Bosque(string nombre, int area)
{
    if (string.IsNullOrWhiteSpace(nombre))
        throw new ArgumentException("El nombre es obligatorio.", nameof(nombre));
    if (area < 0)
        throw new ArgumentOutOfRangeException(nameof(area), "El área no puede ser negativa.");

    Nombre = nombre;
    Area = area;
}
```

Si los datos son inválidos, el objeto **no llega a existir**: `new` lanza la excepción y la variable nunca se asigna.

### El constructor por defecto desaparece

Si una clase **no** declara ningún constructor, el compilador genera uno público y sin parámetros. Por eso funcionaba `new Bosque()` en las lecciones anteriores.

Pero en cuanto declaras **cualquier** constructor, el compilador **deja de generarlo**:

```csharp
class Bosque
{
    public Bosque(string nombre) { /* ... */ }
}

var b = new Bosque();   // error CS7036: no se ha dado ningún argumento que corresponda al parámetro requerido 'nombre'
```

Si quieres permitir ambas formas, declara también el constructor sin parámetros explícitamente.

### `this`: el objeto actual

Dentro de un constructor o de un método de instancia, `this` es una referencia al objeto que se está creando o usando. Su uso más común es **desambiguar** cuando un parámetro se llama igual que un miembro:

```csharp
class Bosque
{
    private string nombre;

    public Bosque(string nombre)
    {
        nombre = nombre;          // ⚠ asigna el parámetro a sí mismo: el campo queda vacío (warning CS1717)
        this.nombre = nombre;     // ✅ this.nombre es el CAMPO; nombre es el PARÁMETRO
    }
}
```

Con la convención de .NET (campos `_nombre`, parámetros `nombre`, propiedades `Nombre`) casi nunca hay ambigüedad, y `this.` es opcional. Algunos equipos lo escriben siempre para que quede claro que es un miembro; otros, nunca. Lo importante es ser consistente.

Otros usos de `this`:

```csharp
public bool EsMayorQue(Bosque otro) => this.Area > otro.Area;   // comparar con otro objeto
public Bosque Clonar() => new Bosque(this.Nombre, this.Area);    // pasarse a sí mismo o usar sus datos
```

`this` no existe en métodos `static`, porque no hay ningún objeto (ver [Miembros estáticos](05-Miembros%20estaticos.md)).

### Sobrecarga de constructores

Como cualquier método, un constructor se puede sobrecargar:

```csharp
class Bosque
{
    public string Nombre { get; }
    public string Pais { get; }

    public Bosque(string nombre, string pais)
    {
        Nombre = nombre;
        Pais = pais;
    }

    public Bosque(string nombre)
    {
        Nombre = nombre;          // ← código duplicado
        Pais = "Desconocido";
    }
}

var b1 = new Bosque("Amazonas", "Brasil");
var b2 = new Bosque("Bosque misterioso");
```

Funciona, pero la asignación de `Nombre` está duplicada. Si mañana agregas una validación, tendrás que acordarte de ponerla en los dos lugares.

### Encadenar constructores con `: this(...)`

Un constructor puede delegar en otro de la misma clase:

```csharp
public Bosque(string nombre, string pais)
{
    Nombre = nombre;
    Pais = pais;
}

public Bosque(string nombre) : this(nombre, "Desconocido")
{
    Console.WriteLine("País no especificado: se usó 'Desconocido'.");
}
```

`: this(nombre, "Desconocido")` ejecuta **primero** el constructor de dos parámetros y **después** el cuerpo del constructor actual. La lógica de asignación vive en un solo lugar.

### Alternativa: un parámetro opcional

Si la única diferencia entre los constructores es un valor por defecto, un parámetro opcional es más simple:

```csharp
public Bosque(string nombre, string pais = "Desconocido")
{
    Nombre = nombre;
    Pais = pais;
}
```

| Técnica | Úsala cuando... |
| --- | --- |
| Parámetro opcional | La única diferencia es un valor por defecto. |
| `: this(...)` | El constructor corto tiene lógica adicional o los tipos de los parámetros son distintos. |

### Orden de inicialización

Al hacer `new Bosque("x") { Area = 5 }`, las cosas pasan en este orden:

1. Los campos se ponen en su valor por defecto (`0`, `null`...).
2. Se ejecutan los **inicializadores de campos y propiedades** (`= valor` en la declaración).
3. Se ejecuta el **constructor** (si hay `: this(...)`, primero el encadenado).
4. Se ejecuta el **inicializador de objeto** (`{ Area = 5 }`).

```text
new Bosque("x") { Area = 5 }

 ① memoria en el heap, campos en default   Nombre = null, Area = 0
            │
 ② inicializadores de campos/propiedades    Area = 1  (si hay "= 1" en la declaración)
            │
 ③ constructor                              : this(...) encadenado primero, luego el cuerpo
            │
 ④ inicializador de objeto                  Area = 5
            │
            ▼
 la variable recibe la referencia al objeto ya terminado
```

Con herencia (se ve en [Herencia](06-Herencia.md)), el paso ③ empieza siempre por el constructor de la clase base y termina en el de la derivada.

```csharp
class Demo
{
    public int Valor { get; set; } = 1;              // paso 2

    public Demo() => Console.WriteLine(Valor);        // paso 3: imprime 1
}

var d = new Demo { Valor = 3 };                        // paso 4: después queda en 3
```

### Constructores primarios (C# 12)

Desde C# 12, los parámetros del constructor se pueden declarar junto al nombre de la clase:

```csharp
var b = new Bosque("Amazonas", 400);
var s = new Saludador("Buenos días");
Console.WriteLine(s.Saludar("Ana"));     // Buenos días, Ana

class Bosque(string nombre, int area)
{
    public string Nombre { get; } = nombre;   // el parámetro inicializa una propiedad
    public int Area { get; } = area;
}

class Saludador(string saludo)
{
    public string Saludar(string persona) => $"{saludo}, {persona}";   // el parámetro se usa en un método
}
```

Detalles importantes:

* `nombre`, `area` y `saludo` **no** son propiedades: son parámetros disponibles en todo el cuerpo de la clase. Si quieres exponerlos, decláralos como propiedades (como `Nombre` arriba).
* Si los usas en métodos (como `saludo`), el compilador los guarda en un campo oculto **mutable**.
* Usar el mismo parámetro para inicializar una propiedad **y** dentro de un método genera `warning CS9124`, porque terminarías con dos copias del dato que pueden divergir. Elige una de las dos formas.
* Cualquier otro constructor de la clase debe encadenar al primario con `: this(...)`.

Son ideales para clases que solo reciben dependencias, por ejemplo servicios con inyección de dependencias: `class PedidoService(IRepositorio repo, ILogger log) { ... }`.

-----

## Ejemplo completo

```csharp
var p1 = new Pedido("Ana", 3, 49.90m);
var p2 = new Pedido("Luis");                       // cantidad y precio por defecto
var p3 = new Pedido("Eva", 1, 120m) { Nota = "Entregar por la tarde" };

foreach (var p in new[] { p1, p2, p3 })
{
    Console.WriteLine(p.Resumen());
}

try
{
    var invalido = new Pedido("", 2, 10m);
}
catch (ArgumentException ex)
{
    Console.WriteLine($"No se creó el pedido: {ex.Message}");
}

class Pedido
{
    private static int _siguienteNumero = 1;

    public int Numero { get; }
    public string Cliente { get; }
    public int Cantidad { get; }
    public decimal PrecioUnitario { get; }
    public DateTime Fecha { get; } = DateTime.Today;    // inicializador de propiedad
    public string Nota { get; init; } = "";

    public decimal Total => Cantidad * PrecioUnitario;

    public Pedido(string cliente, int cantidad, decimal precioUnitario)
    {
        if (string.IsNullOrWhiteSpace(cliente))
            throw new ArgumentException("El cliente es obligatorio.", nameof(cliente));
        if (cantidad <= 0)
            throw new ArgumentOutOfRangeException(nameof(cantidad), "La cantidad debe ser positiva.");

        Numero = _siguienteNumero++;
        Cliente = cliente;
        Cantidad = cantidad;
        PrecioUnitario = precioUnitario;
    }

    public Pedido(string cliente) : this(cliente, 1, 10m)
    {
    }

    public string Resumen()
    {
        string nota = Nota == "" ? "" : $" ({Nota})";
        return $"#{this.Numero} {Cliente}: {Cantidad} x {PrecioUnitario:N2} = {Total:N2}{nota}";
    }
}
```

Salida:

```text
#1 Ana: 3 x 49.90 = 149.70
#2 Luis: 1 x 10.00 = 10.00
#3 Eva: 1 x 120.00 = 120.00 (Entregar por la tarde)
No se creó el pedido: El cliente es obligatorio. (Parameter 'cliente')
```

`_siguienteNumero` es `static`: un solo contador compartido por todos los pedidos. Se explica en la [próxima lección](05-Miembros%20estaticos.md).

-----

## Errores comunes

**1. Llamar a `new Clase()` cuando solo existen constructores con parámetros.**
Qué pasa: `error CS7036: There is no argument given that corresponds to the required parameter 'nombre' of 'Bosque.Bosque(string)'`.
Por qué: al declarar un constructor, el compilador deja de generar el constructor sin parámetros.
Arreglo: pasa los argumentos o declara un constructor sin parámetros.

**2. Ponerle tipo de retorno al constructor.**
Qué pasa: `public void Bosque() { }` dentro de `class Bosque` da `error CS0542: 'Bosque': member names cannot be the same as their enclosing type`.
Por qué: con tipo de retorno, el compilador lo interpreta como un **método** normal, y un método no puede llamarse igual que su clase.
Arreglo: quita el tipo de retorno: `public Bosque() { }`.

**3. Asignar un parámetro a sí mismo.**
Qué pasa: `warning CS1717: Assignment made to same variable; did you mean to assign something else?` y el campo queda sin valor.
Por qué: `nombre = nombre;` se refiere al parámetro en los dos lados.
Arreglo: `this.nombre = nombre;` o usa la convención `_nombre`.

**4. Duplicar lógica entre constructores.**
Qué pasa: no hay error, pero una validación queda en un constructor y no en otro.
Por qué: copiar y pegar entre sobrecargas.
Arreglo: encadena con `: this(...)` o usa parámetros opcionales.

**5. Llamar a métodos virtuales en el constructor.**
Qué pasa: una clase hija recibe la llamada antes de que su propio constructor haya inicializado sus datos.
Por qué: el constructor de la clase base se ejecuta primero (ver [Herencia](06-Herencia.md)).
Arreglo: no llames métodos `virtual` desde los constructores.

-----

## Según la versión de C#

* **C# 6:** inicializadores de propiedades autoimplementadas (`{ get; } = valor`).
* **C# 7:** constructores con cuerpo de expresión (`public Bosque(string n) => Nombre = n;`).
* **C# 9:** `new()` con tipo de destino, `init` y records con constructor posicional.
* **C# 11:** `required`, que complementa (o reemplaza) a los constructores cuando se usan inicializadores de objetos.
* **C# 12:** constructores primarios en clases y structs.

-----

## Cuándo sí y cuándo no

**Usa un constructor con parámetros cuando:**

* Hay datos **obligatorios** sin los cuales el objeto no tiene sentido.
* Hay reglas que deben cumplirse desde la creación.

**Usa inicializadores de objetos (`{ get; init; }`, `required`) cuando:**

* Hay muchos datos, varios opcionales, y un constructor con 8 parámetros sería ilegible.

**Usa un constructor primario cuando:**

* La clase solo recibe dependencias o datos y los usa internamente.

**Evita:**

* Constructores que hacen trabajo pesado (leer archivos, llamar a APIs): un constructor debe ser rápido y predecible.
* Muchas sobrecargas de constructores: con más de tres, considera un método de fábrica o un *builder* (ver [Encadenamiento de métodos](11-Encadenamiento%20de%20metodos.md)).

-----

## Resumen en 5 líneas

1. Un constructor se llama igual que la clase, no tiene tipo de retorno y se ejecuta con `new`.
2. Úsalo para exigir los datos obligatorios y validar: el objeto nace válido o no nace.
3. Si declaras cualquier constructor, desaparece el constructor sin parámetros automático.
4. `this` es el objeto actual; `this.campo = campo` desambigua.
5. `: this(...)` encadena constructores y evita duplicar código; desde C# 12 existen los constructores primarios.

-----

## Para profundizar

<details>
<summary>Métodos de fábrica estáticos</summary>

A veces un constructor no alcanza para expresar la intención, sobre todo cuando hay dos formas de crear un objeto con los mismos tipos de parámetros:

```csharp
class Temperatura
{
    public double Kelvin { get; }
    private Temperatura(double kelvin) => Kelvin = kelvin;   // constructor privado

    public static Temperatura DesdeCelsius(double c) => new(c + 273.15);
    public static Temperatura DesdeFahrenheit(double f) => new((f - 32) * 5 / 9 + 273.15);
}

var t = Temperatura.DesdeCelsius(25);
```

Con un constructor privado y métodos estáticos con nombre, la intención queda clara y no hay ambigüedad. Se usa mucho en .NET: `TimeSpan.FromMinutes`, `Guid.NewGuid`, `DateTime.Parse`.

</details>

<details>
<summary>Deconstructores</summary>

Un método `Deconstruct` permite descomponer un objeto en variables, igual que una tupla:

```csharp
class Punto
{
    public int X { get; }
    public int Y { get; }
    public Punto(int x, int y) => (X, Y) = (x, y);       // asignación con tupla

    public void Deconstruct(out int x, out int y) => (x, y) = (X, Y);
}

var (x, y) = new Punto(3, 4);
```

Los records lo generan automáticamente para sus parámetros posicionales.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un constructor es un método especial con el nombre de la clase y sin tipo de retorno, que se ejecuta al crear un objeto con `new`. Sirve para inicializar sus datos y validarlos. Si no declaras ninguno, C# genera uno sin parámetros. `this` es una referencia al objeto actual y se usa, por ejemplo, para diferenciar un campo de un parámetro con el mismo nombre.

### Respuesta ampliada (semi-senior)

El constructor garantiza las invariantes desde la creación: si lanza una excepción, el objeto no queda accesible. El compilador solo genera el constructor sin parámetros si no hay ningún otro. El orden es: valores por defecto, inicializadores de campos, constructor de la base (con herencia), cuerpo del constructor y, por último, el inicializador de objeto, que se ejecuta después del constructor, por lo que este no puede validar propiedades asignadas con `init` (para eso existe `required` o la validación en el propio accesor). `: this(...)` centraliza la lógica entre sobrecargas. Los constructores primarios de C# 12 capturan los parámetros como estado mutable oculto si se usan fuera de los inicializadores. Para creaciones con semántica distinta se usan métodos de fábrica estáticos con constructor privado.

### Preguntas frecuentes de seguimiento

**1. ¿Una clase puede no tener constructor?**
No en la práctica: si no declaras ninguno, el compilador genera uno público y sin parámetros. Las clases `static` sí no tienen constructores de instancia.

**2. ¿Un constructor puede ser privado?**
Sí. Se usa para obligar a crear objetos mediante métodos de fábrica estáticos, o en el patrón Singleton.

**3. ¿Qué se ejecuta primero, el constructor o el inicializador de objeto?**
El constructor. El inicializador de objeto (`{ Prop = valor }`) asigna después.

-----

## Práctica

**Ejercicio 1.** Crea una clase `Rectangulo` con `Ancho` y `Alto` de solo lectura. Debe tener un constructor que reciba ambos (y que rechace valores ≤ 0) y otro que reciba un solo valor para crear un cuadrado, encadenado al primero. Agrega una propiedad calculada `Area`.

<details>
<summary>Solución</summary>

```csharp
var r = new Rectangulo(3, 4);
var c = new Rectangulo(5);
Console.WriteLine($"{r.Area} {c.Area}");   // 12 25

class Rectangulo
{
    public double Ancho { get; }
    public double Alto { get; }
    public double Area => Ancho * Alto;

    public Rectangulo(double ancho, double alto)
    {
        if (ancho <= 0 || alto <= 0)
            throw new ArgumentOutOfRangeException(nameof(ancho), "Las medidas deben ser positivas.");
        Ancho = ancho;
        Alto = alto;
    }

    public Rectangulo(double lado) : this(lado, lado) { }
}
```

</details>

**Ejercicio 2.** ¿Qué imprime este código? Piensa en el orden de inicialización.

```csharp
var x = new Caja(5) { Valor = 9 };
Console.WriteLine(x.Valor);

class Caja
{
    public int Valor { get; set; } = 1;

    public Caja(int v)
    {
        Console.WriteLine(Valor);
        Valor = v;
        Console.WriteLine(Valor);
    }
}
```

<details>
<summary>Solución</summary>

```text
1
5
9
```

Primero el inicializador de la propiedad (1), luego el constructor (imprime 1, asigna 5, imprime 5) y, por último, el inicializador de objeto (9).

</details>

-----

## Siguiente lección

[Miembros estáticos](05-Miembros%20estaticos.md)
