# Encadenamiento de métodos

## En una frase

Encadenar métodos (*method chaining*) es llamar varios métodos seguidos en una sola expresión, `objeto.A().B().C()`, algo posible porque cada método **devuelve un objeto** (el mismo con `return this`, o uno nuevo) sobre el que se llama al siguiente; es la base de las **APIs fluidas** y del patrón **Builder**.

-----

## Antes de empezar

Conviene que ya sepas:

* Métodos de instancia que devuelven valores y la palabra clave `this`, de [Constructores y this](04-Constructores%20y%20this.md).
* Que los strings son inmutables y sus métodos devuelven strings nuevos, de [Métodos de string y StringBuilder](../01-tipos-y-variables/06-Metodos%20de%20string%20y%20StringBuilder.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Encadenamiento de métodos:** llamar a un método sobre el resultado del anterior, en una misma expresión.
* **API fluida (*fluent API*):** API diseñada para leerse como una frase mediante el encadenamiento.
* **`return this`:** devolver el propio objeto para permitir el encadenamiento.
* **Patrón Builder:** objeto auxiliar que construye otro objeto paso a paso y lo entrega con un método final (`Build`, `Construir`).
* **Inmutable:** objeto que no cambia; cada operación devuelve uno nuevo.

-----

## El problema

Tienes una calculadora con estado y quieres hacer `(5 + 3) × 2`:

```csharp
var calc = new Calculadora(5);
calc.Sumar(3);
calc.Multiplicar(2);
var resultado = calc.Resultado();
```

Cuatro líneas y una variable intermedia para una sola operación. Y si construyes un objeto con muchos datos opcionales, el constructor acaba con diez parámetros (`new Usuario("Ana", null, 30, null, true, false, ...)`) imposibles de leer.

Lo que te gustaría escribir es algo que se lea como una frase:

```csharp
var resultado = new Calculadora(5).Sumar(3).Multiplicar(2).Resultado();
```

-----

## Cómo funciona

### Ya lo usaste: encadenar sobre el valor devuelto

Cada método devuelve un valor, y sobre ese valor puedes llamar al siguiente método:

```csharp
string limpio = "  Hola Mundo  ".Trim().ToUpper().Replace("MUNDO", "C#");
// "  Hola Mundo  " → Trim() → "Hola Mundo" → ToUpper() → "HOLA MUNDO" → Replace → "HOLA C#"
```

Aquí cada método devuelve un **string nuevo** (los strings son inmutables). El encadenamiento no es una sintaxis especial: es simplemente llamar a un método sobre lo que devolvió el anterior.

### Encadenar con `return this`

Para que **tus** métodos se puedan encadenar, haz que devuelvan el propio objeto:

```csharp
class Calculadora
{
    private double _valor;

    public Calculadora(double valorInicial = 0) => _valor = valorInicial;

    public Calculadora Sumar(double n)
    {
        _valor += n;
        return this;            // 🔑 devuelve el mismo objeto
    }

    public Calculadora Multiplicar(double n)
    {
        _valor *= n;
        return this;
    }

    public double Resultado() => _valor;    // método "terminal": devuelve el dato final
}
```

```csharp
double r = new Calculadora(5)
    .Sumar(3)          // 5 + 3 = 8
    .Multiplicar(2)    // 8 × 2 = 16
    .Resultado();      // 16

Console.WriteLine(r);  // 16
```

* El tipo de retorno de `Sumar` y `Multiplicar` es `Calculadora`, no `void`.
* `return this` entrega el mismo objeto sobre el que se llamó.
* Un método **terminal** (`Resultado`) corta la cadena y devuelve el dato final.
* Por convención, cuando la cadena es larga, se escribe **un método por línea** con el punto al principio.

### Encadenamiento inmutable: devolver un objeto nuevo

En lugar de modificar `this`, cada método puede devolver un objeto **nuevo**. Así funcionan `string`, `DateTime` y LINQ:

```csharp
DateTime vencimiento = DateTime.Today.AddMonths(1).AddDays(-1);   // DateTime.Today no cambia

var nombres = new[] { "ana", "luis", "eva", "beatriz" }
    .Where(n => n.Length > 3)
    .Select(n => n.ToUpper())
    .OrderBy(n => n)
    .ToList();                    // [BEATRIZ, LUIS]
```

| Estilo | Cómo | Ventaja | Riesgo |
| --- | --- | --- | --- |
| Mutable (`return this`) | Modifica el objeto y lo devuelve | Sin objetos intermedios | El objeto cambia: si otra variable lo referencia, también ve los cambios |
| Inmutable (`return new ...`) | Devuelve una copia modificada | Seguro de compartir | Hay que **guardar** el resultado o se pierde |

### El patrón Builder

Cuando un objeto tiene muchos datos (varios opcionales), un *builder* permite construirlo paso a paso con nombres claros:

```csharp
class Usuario
{
    public string Nombre { get; }
    public int Edad { get; }
    public string? Correo { get; }
    public bool EsAdmin { get; }

    // Constructor interno: solo el builder lo usa
    internal Usuario(string nombre, int edad, string? correo, bool esAdmin)
    {
        Nombre = nombre;
        Edad = edad;
        Correo = correo;
        EsAdmin = esAdmin;
    }

    public override string ToString() =>
        $"{Nombre}, {Edad} años, {Correo ?? "sin correo"}{(EsAdmin ? ", admin" : "")}";
}

class UsuarioBuilder
{
    private string _nombre = "";
    private int _edad;
    private string? _correo;
    private bool _esAdmin;

    public UsuarioBuilder ConNombre(string nombre) { _nombre = nombre; return this; }
    public UsuarioBuilder ConEdad(int edad) { _edad = edad; return this; }
    public UsuarioBuilder ConCorreo(string correo) { _correo = correo; return this; }
    public UsuarioBuilder ComoAdmin() { _esAdmin = true; return this; }

    public Usuario Construir()
    {
        if (string.IsNullOrWhiteSpace(_nombre))
            throw new InvalidOperationException("El nombre es obligatorio.");
        return new Usuario(_nombre, _edad, _correo, _esAdmin);
    }
}
```

```csharp
var admin = new UsuarioBuilder()
    .ConNombre("Carlos")
    .ConEdad(24)
    .ConCorreo("carlos@correo.com")
    .ComoAdmin()
    .Construir();

var invitado = new UsuarioBuilder()
    .ConNombre("Invitado")
    .Construir();                      // los opcionales simplemente no se llaman

Console.WriteLine(admin);      // Carlos, 24 años, carlos@correo.com, admin
Console.WriteLine(invitado);   // Invitado, 0 años, sin correo
```

Ventajas frente a un constructor con muchos parámetros:

* Cada dato tiene un **nombre** en la llamada.
* Los opcionales se omiten sin pasar `null`.
* La validación se hace **una vez**, al final, en `Construir()`.
* El objeto final (`Usuario`) puede ser inmutable.

### APIs fluidas que vas a usar

```csharp
// StringBuilder: Append devuelve el mismo StringBuilder
var html = new StringBuilder()
    .Append("<ul>")
    .Append("<li>Uno</li>")
    .Append("</ul>")
    .ToString();

// LINQ: cada operador devuelve una nueva secuencia
var top3 = productos.Where(p => p.Stock > 0).OrderByDescending(p => p.Ventas).Take(3);

// ASP.NET Core: configuración fluida
builder.Services.AddControllers().AddJsonOptions(o => o.JsonSerializerOptions.WriteIndented = true);

// Entity Framework Core
var pedidos = db.Pedidos.Where(p => p.ClienteId == id).Include(p => p.Items).ToList();
```

-----

## Ejemplo completo

Un constructor de consultas SQL simplificado (con fines didácticos: en producción, usa parámetros y nunca concatenes valores del usuario):

```csharp
string sql = new ConsultaBuilder("Productos")
    .Seleccionar("Id", "Nombre", "Precio")
    .Donde("Precio > 100")
    .Donde("Stock > 0")
    .OrdenarPor("Precio", descendente: true)
    .Limitar(10)
    .Construir();

Console.WriteLine(sql);

string simple = new ConsultaBuilder("Clientes").Construir();
Console.WriteLine(simple);

class ConsultaBuilder
{
    private readonly string _tabla;
    private readonly List<string> _columnas = new();
    private readonly List<string> _condiciones = new();
    private string? _orden;
    private int? _limite;

    public ConsultaBuilder(string tabla) => _tabla = tabla;

    public ConsultaBuilder Seleccionar(params string[] columnas)
    {
        _columnas.AddRange(columnas);
        return this;
    }

    public ConsultaBuilder Donde(string condicion)
    {
        _condiciones.Add(condicion);
        return this;
    }

    public ConsultaBuilder OrdenarPor(string columna, bool descendente = false)
    {
        _orden = $"{columna} {(descendente ? "DESC" : "ASC")}";
        return this;
    }

    public ConsultaBuilder Limitar(int cantidad)
    {
        if (cantidad <= 0) throw new ArgumentOutOfRangeException(nameof(cantidad));
        _limite = cantidad;
        return this;
    }

    public string Construir()
    {
        var sb = new StringBuilder();
        sb.Append("SELECT ")
          .Append(_columnas.Count > 0 ? string.Join(", ", _columnas) : "*")
          .Append(" FROM ").Append(_tabla);

        if (_condiciones.Count > 0)
            sb.Append(" WHERE ").Append(string.Join(" AND ", _condiciones));
        if (_orden is not null)
            sb.Append(" ORDER BY ").Append(_orden);
        if (_limite is not null)
            sb.Append(" LIMIT ").Append(_limite);

        return sb.ToString();
    }
}
```

Para que compile, agrega `using System.Text;` al principio del archivo. Salida:

```text
SELECT Id, Nombre, Precio FROM Productos WHERE Precio > 100 AND Stock > 0 ORDER BY Precio DESC LIMIT 10
SELECT * FROM Clientes
```

El builder usa encadenamiento hacia afuera (`return this`) y, por dentro, el encadenamiento de `StringBuilder`.

-----

## Errores comunes

**1. Un método `void` en medio de la cadena.**
Qué pasa: `new Calculadora(5).Reiniciar().Sumar(3)` da `error CS0023: Operator '.' cannot be applied to operand of type 'void'`.
Por qué: un método `void` no devuelve nada sobre lo cual llamar al siguiente.
Arreglo: haz que devuelva el objeto (`return this;`) si debe formar parte de la API fluida.

**2. No guardar el resultado de una cadena inmutable.**
Qué pasa: `fecha.AddDays(1).AddHours(2);` no cambia `fecha`.
Por qué: `DateTime` es inmutable: la cadena produjo un valor nuevo que se descartó.
Arreglo: `fecha = fecha.AddDays(1).AddHours(2);`.

**3. Reutilizar un builder mutable sin querer.**
Qué pasa: dos objetos construidos con el mismo builder comparten datos que no debían (el segundo hereda lo que se configuró para el primero).
Por qué: `return this` devuelve el mismo builder, con su estado acumulado.
Arreglo: crea un builder nuevo por objeto, o diseña `Construir()` para que reinicie el estado.

**4. `null` en mitad de la cadena.**
Qué pasa: `NullReferenceException` en una línea con cinco llamadas, y no sabes cuál falló.
Por qué: algún método devolvió `null`.
Arreglo: los métodos fluidos nunca deberían devolver `null`. Si una parte puede serlo, usa `?.` o divide la cadena en variables para depurar.

**5. Cadenas que atraviesan muchos objetos ajenos.**
Qué pasa: `pedido.GetCliente().GetDireccion().GetCiudad().GetNombre()` compila, pero acopla tu código a la estructura interna de cuatro clases.
Por qué: esto no es una API fluida, es "navegar" por objetos (viola la Ley de Demeter).
Arreglo: pide lo que necesitas a quien lo sabe: `pedido.CiudadDeEntrega()`.

-----

## Según la versión de C#

El encadenamiento existe desde C# 1; lo que cambió es cómo se combina con otras características:

* **C# 3:** LINQ y los métodos de extensión popularizaron las APIs fluidas.
* **C# 6:** el operador `?.` permite encadenar sobre valores que pueden ser `null`.
* **C# 9:** los `record` y la expresión `with` ofrecen "copias modificadas" sin escribir un builder: `var b = a with { Edad = 30 };`.
* **C# 11:** `required` + `init` con inicializadores de objeto cubren muchos casos en los que antes se usaba un builder.

-----

## Cuándo sí y cuándo no

**Usa encadenamiento con `return this` cuando:**

* Diseñas una API de configuración o construcción que se lee mejor como una frase.

**Usa un builder cuando:**

* El objeto tiene muchos parámetros, varios opcionales, y validaciones que dependen de la combinación.
* Quieres que el objeto final sea inmutable.

**Prefiere un inicializador de objeto (`new X { A = 1, B = 2 }` con `init`/`required`) cuando:**

* Solo asignas propiedades sin lógica de construcción. Es menos código que un builder.

**Evita:**

* Cadenas muy largas sin saltos de línea: son difíciles de leer y de depurar.
* Mezclar en una misma cadena métodos que modifican y métodos que consultan, porque dificulta entender el estado.

-----

## Resumen en 5 líneas

1. Encadenar es llamar a un método sobre lo que devolvió el anterior: `a.B().C()`.
2. Para encadenar tus métodos, devuelve el objeto: `return this;` (mutable) o un objeto nuevo (inmutable).
3. Un método terminal (`Resultado()`, `Construir()`, `ToList()`) cierra la cadena.
4. El patrón Builder construye objetos complejos paso a paso con nombres claros y valida al final.
5. Con objetos inmutables (`string`, `DateTime`), guarda el resultado de la cadena o se pierde.

-----

## Para profundizar

<details>
<summary>Builder inmutable</summary>

Un builder también puede ser inmutable: cada método devuelve un builder **nuevo**. Así se puede reutilizar una configuración base sin efectos secundarios:

```csharp
record ConsultaInmutable(string Tabla, int? Limite = null, string? Orden = null)
{
    public ConsultaInmutable Limitar(int n) => this with { Limite = n };
    public ConsultaInmutable OrdenarPor(string c) => this with { Orden = c };
}

var basica = new ConsultaInmutable("Productos").OrdenarPor("Nombre");
var top10 = basica.Limitar(10);   // basica no cambió
var top5 = basica.Limitar(5);
```

Es el estilo de las configuraciones de algunas librerías modernas y de LINQ.

</details>

<details>
<summary>Ley de Demeter: fluido no es lo mismo que "navegar"</summary>

La Ley de Demeter ("habla solo con tus amigos cercanos") dice que un método debería llamar solo a métodos de su propio objeto, de sus parámetros o de los objetos que crea. `a.GetB().GetC().Hacer()` la viola, porque tu código conoce la estructura interna de `a` y de `b`. Una API fluida **no** la viola: todas las llamadas son sobre el mismo objeto (o el mismo tipo de objeto) diseñado para eso.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El encadenamiento de métodos consiste en llamar varios métodos seguidos en una sola expresión, porque cada método devuelve el objeto sobre el que se llama al siguiente, normalmente con `return this`. Se usa en las APIs fluidas como LINQ o `StringBuilder`, y en el patrón Builder para construir objetos paso a paso.

### Respuesta ampliada (semi-senior)

Hay dos variantes: la mutable, que devuelve `this` (como `StringBuilder`, o los builders clásicos), y la inmutable, que devuelve una nueva instancia (como `string`, `DateTime`, LINQ o los records con `with`). La mutable evita asignaciones, pero introduce estado compartido; la inmutable es más segura de compartir y reutilizar. El patrón Builder separa la construcción de la representación, permite validar combinaciones antes de crear el objeto y deja el resultado inmutable; en C# moderno, `required`/`init` cubren muchos casos simples. Hay que distinguir una API fluida diseñada para encadenar de las cadenas que atraviesan el grafo de objetos, que violan la Ley de Demeter.

### Preguntas frecuentes de seguimiento

**1. ¿Qué tiene que devolver un método para poder encadenarse?**
Un objeto: el mismo (`this`) o uno nuevo del tipo sobre el que se quiere seguir llamando.

**2. ¿Cuándo usarías un Builder en lugar de un constructor?**
Cuando hay muchos parámetros, varios opcionales, o validaciones que dependen de la combinación de valores.

**3. ¿LINQ modifica la colección original al encadenar?**
No. Cada operador devuelve una nueva secuencia (y además se evalúa de forma diferida hasta que se recorre).

-----

## Práctica

**Ejercicio 1.** Crea una clase `Texto` con un campo privado `string` y métodos encadenables `Mayusculas()`, `Recortar()`, `Reemplazar(string viejo, string nuevo)` y `AgregarAlFinal(string s)`, más un método terminal `Obtener()`. Úsala para convertir `"  hola mundo  "` en `"HOLA C#!"`.

<details>
<summary>Solución</summary>

```csharp
string resultado = new Texto("  hola mundo  ")
    .Recortar()
    .Mayusculas()
    .Reemplazar("MUNDO", "C#")
    .AgregarAlFinal("!")
    .Obtener();

Console.WriteLine(resultado);   // HOLA C#!

class Texto
{
    private string _valor;
    public Texto(string valor) => _valor = valor;

    public Texto Mayusculas() { _valor = _valor.ToUpper(); return this; }
    public Texto Recortar() { _valor = _valor.Trim(); return this; }
    public Texto Reemplazar(string viejo, string nuevo) { _valor = _valor.Replace(viejo, nuevo); return this; }
    public Texto AgregarAlFinal(string s) { _valor += s; return this; }

    public string Obtener() => _valor;
}
```

</details>

**Ejercicio 2.** Crea un `PizzaBuilder` con los métodos `Tamano(string)`, `ConIngrediente(string)` (se puede llamar varias veces) y `ConBordeRelleno()`, y un `Construir()` que lance una excepción si no se eligió tamaño. La pizza resultante debe tener un `ToString()` como `Pizza grande con queso, jamón (borde relleno)`.

<details>
<summary>Solución</summary>

```csharp
var pizza = new PizzaBuilder()
    .Tamano("grande")
    .ConIngrediente("queso")
    .ConIngrediente("jamón")
    .ConBordeRelleno()
    .Construir();

Console.WriteLine(pizza);   // Pizza grande con queso, jamón (borde relleno)

class Pizza
{
    public string Tamano { get; }
    public IReadOnlyList<string> Ingredientes { get; }
    public bool BordeRelleno { get; }

    public Pizza(string tamano, List<string> ingredientes, bool bordeRelleno)
    {
        Tamano = tamano;
        Ingredientes = ingredientes.ToArray();   // copia: la pizza no depende de la lista del builder
        BordeRelleno = bordeRelleno;
    }

    public override string ToString()
    {
        string ingredientes = Ingredientes.Count > 0 ? $" con {string.Join(", ", Ingredientes)}" : "";
        string borde = BordeRelleno ? " (borde relleno)" : "";
        return $"Pizza {Tamano}{ingredientes}{borde}";
    }
}

class PizzaBuilder
{
    private string? _tamano;
    private readonly List<string> _ingredientes = new();
    private bool _bordeRelleno;

    public PizzaBuilder Tamano(string tamano) { _tamano = tamano; return this; }
    public PizzaBuilder ConIngrediente(string ingrediente) { _ingredientes.Add(ingrediente); return this; }
    public PizzaBuilder ConBordeRelleno() { _bordeRelleno = true; return this; }

    public Pizza Construir() =>
        _tamano is null
            ? throw new InvalidOperationException("Debes elegir un tamaño.")
            : new Pizza(_tamano, _ingredientes, _bordeRelleno);
}
```

</details>

-----

## Siguiente lección

Terminaste el módulo de programación orientada a objetos. Continúa con [Tipos avanzados](../05-tipos-avanzados/README.md).
