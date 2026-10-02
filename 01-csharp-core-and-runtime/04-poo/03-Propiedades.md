# Propiedades

## En una frase

Una propiedad se usa como un campo (`bosque.Area = 10`), pero por dentro ejecuta código: un accesor `get` al leer y un accesor `set` al escribir, lo que permite **validar**, **calcular** o **restringir** el acceso a los datos de un objeto.

-----

## Antes de empezar

Conviene que ya sepas:

* Hacer campos privados y entender la encapsulación, de [Modificadores de acceso y encapsulación](02-Modificadores%20de%20acceso%20y%20encapsulacion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Propiedad:** miembro que se usa como un campo pero ejecuta código al leerse o escribirse.
* **Accesor `get` (*getter*):** bloque que se ejecuta al leer la propiedad.
* **Accesor `set` (*setter*):** bloque que se ejecuta al asignar la propiedad.
* **`value`:** palabra clave que, dentro del `set`, representa el valor que se está asignando.
* **Campo de respaldo (*backing field*):** el campo privado donde la propiedad guarda realmente el dato.
* **Propiedad autoimplementada:** propiedad cuyo campo de respaldo genera el compilador: `{ get; set; }`.
* **Propiedad calculada:** propiedad que no guarda un dato, sino que lo calcula a partir de otros.
* **`init`:** accesor que permite asignar la propiedad solo al crear el objeto.

-----

## El problema

En la lección anterior encapsulaste los datos con métodos:

```csharp
bosque.EstablecerArea(400);
Console.WriteLine(bosque.ObtenerArea());
```

Funciona, pero tiene dos problemas:

1. Es verboso: un par de métodos por cada dato.
2. Es menos natural que `bosque.Area = 400;`, que es como se trabaja con cualquier otro dato.

Si haces el campo público para escribir `bosque.Area = -1249`, pierdes la validación. Las **propiedades** dan lo mejor de los dos mundos: la sintaxis de un campo con el control de un método.

-----

## Cómo funciona

### Una propiedad completa

```csharp
class Bosque
{
    private int _area;              // campo de respaldo: privado

    public int Area                 // propiedad: pública
    {
        get { return _area; }       // se ejecuta al LEER
        set { _area = value; }      // se ejecuta al ESCRIBIR; value = lo que se asigna
    }
}
```

```csharp
var b = new Bosque();
b.Area = 400;                 // llama a set, con value = 400
Console.WriteLine(b.Area);    // llama a get → 400
```

* Los accesores `get` y `set` **no** llevan paréntesis.
* Convención: el campo se llama `_area` y la propiedad `Area` (PascalCase).
* Para quien usa la clase, `Area` parece un campo. Por dentro, son dos métodos.

### Validación en el `set`

Aquí está el valor real de las propiedades:

```csharp
class Bosque
{
    private int _area;

    public int Area
    {
        get => _area;
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "El área no puede ser negativa.");
            _area = value;
        }
    }
}
```

```csharp
var b = new Bosque();
b.Area = 400;     // ✅
b.Area = -1;      // ❌ ArgumentOutOfRangeException: el objeto nunca queda en un estado inválido
```

Ante un valor inválido tienes dos opciones:

| Opción | Ejemplo | Ventaja | Riesgo |
| --- | --- | --- | --- |
| **Lanzar una excepción** | `throw new ArgumentOutOfRangeException(...)` | El error se detecta en el momento | Hay que manejarla |
| **Corregir el valor** | `_area = value < 0 ? 0 : value;` | Nunca falla | Esconde errores: quien asignó -1 no se entera |

Por defecto, **prefiere lanzar una excepción**: un dato inválido casi siempre es un bug que conviene ver cuanto antes. Corregir en silencio solo tiene sentido cuando la regla de negocio lo pide de forma explícita (por ejemplo, limitar un volumen entre 0 y 100).

### Propiedades autoimplementadas

Si el `get` y el `set` solo leen y escriben el campo, sin lógica, no hace falta escribirlos:

```csharp
class Bosque
{
    public string Nombre { get; set; } = "";     // con valor inicial
    public int Arboles { get; set; }
}
```

El compilador crea un campo de respaldo oculto. Es la forma más común de declarar datos en una clase.

¿Y por qué no un campo público, si hace lo mismo? Porque una propiedad se puede cambiar después (agregarle validación, hacer el `set` privado) **sin cambiar el código de quien la usa**, y porque gran parte de .NET (serialización JSON, enlace de datos, Entity Framework) trabaja con propiedades, no con campos.

### Controlar quién puede escribir

**Solo lectura (sin `set`):** se asigna solo en el constructor o en la declaración.

```csharp
class Bosque
{
    public string Pais { get; }                 // solo get

    public Bosque(string pais) => Pais = pais;  // ✅ en el constructor sí se puede
}

var b = new Bosque("Perú");
b.Pais = "Chile";   // error CS0200: la propiedad o el indizador 'Bosque.Pais' no se puede asignar (es de solo lectura)
```

**`set` privado:** la clase puede modificarla; desde afuera solo se lee.

```csharp
class Bosque
{
    public int Arboles { get; private set; }

    public void Plantar(int cantidad) => Arboles += cantidad;   // ✅ dentro de la clase
}

var b = new Bosque();
b.Plantar(30);
b.Arboles = 100;    // error CS0272: el accesor set es inaccesible
```

**`init`:** se puede asignar al crear el objeto (en el inicializador) y después ya no:

```csharp
class Producto
{
    public string Codigo { get; init; } = "";
}

var p = new Producto { Codigo = "A-001" };   // ✅ al crear
p.Codigo = "B-002";                           // error CS8852: solo se puede asignar en un inicializador de objeto
```

**`required`:** obliga a asignar la propiedad al crear el objeto:

```csharp
class Usuario
{
    public required string Email { get; init; }
}

var u1 = new Usuario { Email = "ana@mail.com" };   // ✅
var u2 = new Usuario();                             // error CS9035: el miembro requerido 'Usuario.Email' debe establecerse
```

| Declaración | Leer desde fuera | Escribir desde fuera | Escribir dentro de la clase |
| --- | --- | --- | --- |
| `{ get; set; }` | Sí | Sí | Sí |
| `{ get; private set; }` | Sí | No | Sí |
| `{ get; init; }` | Sí | Solo al crear | Solo al crear o en el constructor |
| `{ get; }` | Sí | No | Solo en el constructor |

### Propiedades calculadas

Una propiedad no tiene por qué guardar un dato: puede **calcularlo** cada vez que se lee.

```csharp
class Rectangulo
{
    public double Ancho { get; set; }
    public double Alto { get; set; }

    public double Area => Ancho * Alto;              // solo get, con cuerpo de expresión
    public bool EsCuadrado => Ancho == Alto;
}

var r = new Rectangulo { Ancho = 3, Alto = 4 };
Console.WriteLine(r.Area);   // 12
r.Ancho = 4;
Console.WriteLine(r.Area);   // 16: siempre está actualizada
```

`public double Area => Ancho * Alto;` es una propiedad de solo lectura con cuerpo de expresión. No guarda nada: no puede quedar desactualizada.

### Accesores con cuerpo de expresión

```csharp
private string _nombre = "";

public string Nombre
{
    get => _nombre;
    set => _nombre = value.Trim();
}
```

### La palabra clave `field` (C# 14)

Antes, para agregar una validación había que renunciar a la propiedad autoimplementada y escribir el campo a mano. Desde C# 14, `field` se refiere al campo de respaldo que genera el compilador:

```csharp
public int Area
{
    get;
    set => field = value >= 0 ? value : throw new ArgumentOutOfRangeException(nameof(value));
}
```

Si trabajas con una versión anterior, verás la forma con `private int _area;` explícito.

-----

## Ejemplo completo

```csharp
var empleado = new Empleado("E-001")
{
    Nombre = "  ana torres  ",
    SueldoMensual = 3500m
};

empleado.RegistrarHorasExtra(10);

Console.WriteLine($"[{empleado.Codigo}] {empleado.Nombre}");
Console.WriteLine($"Sueldo anual: {empleado.SueldoAnual:N2}");
Console.WriteLine($"Horas extra: {empleado.HorasExtra}");
Console.WriteLine($"¿Sueldo alto? {empleado.EsSueldoAlto}");

try
{
    empleado.SueldoMensual = -100;
}
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}

class Empleado
{
    private string _nombre = "";
    private decimal _sueldoMensual;

    public Empleado(string codigo) => Codigo = codigo;

    public string Codigo { get; }                       // solo lectura: lo fija el constructor

    public string Nombre                                // normaliza el texto al asignarlo
    {
        get => _nombre;
        set => _nombre = System.Globalization.CultureInfo.CurrentCulture.TextInfo
                            .ToTitleCase(value.Trim().ToLower());
    }

    public decimal SueldoMensual                        // valida
    {
        get => _sueldoMensual;
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "El sueldo no puede ser negativo.");
            _sueldoMensual = value;
        }
    }

    public int HorasExtra { get; private set; }         // solo la clase la modifica

    public decimal SueldoAnual => SueldoMensual * 12;   // calculada
    public bool EsSueldoAlto => SueldoMensual > 5000m;  // calculada

    public void RegistrarHorasExtra(int horas)
    {
        if (horas > 0) HorasExtra += horas;
    }
}
```

Salida:

```text
[E-001] Ana Torres
Sueldo anual: 42,000.00
Horas extra: 10
¿Sueldo alto? False
Error: El sueldo no puede ser negativo. (Parameter 'value')
```

Cada propiedad usa la herramienta adecuada: solo lectura, normalización, validación, `set` privado y cálculo. `try/catch` captura la excepción; las excepciones se estudian en [Manejo de excepciones](../08-excepciones/01-Manejo%20de%20excepciones.md).

-----

## Errores comunes

**1. Asignar una propiedad de solo lectura.**
Qué pasa: `error CS0200: Property or indexer 'Bosque.Pais' cannot be assigned to -- it is read only`.
Por qué: la propiedad no tiene `set`.
Arreglo: asígnala en el constructor, o agrega `init` o `private set` según lo que necesites.

**2. Asignar una propiedad con `set` privado desde fuera.**
Qué pasa: `error CS0272: The property or indexer 'Bosque.Arboles' cannot be used in this context because the set accessor is inaccessible`.
Por qué: solo la clase puede usar ese `set`.
Arreglo: usa el método que la clase ofrece para modificarla.

**3. Recursión infinita en una propiedad.**
Qué pasa: `public int Area { get => Area; set => Area = value; }` compila, pero al usarla lanza `StackOverflowException` y el proceso termina.
Por qué: la propiedad se llama a sí misma en lugar de usar el campo.
Arreglo: usa el campo de respaldo (`_area`) o `field`.

**4. Tipo de la propiedad distinto al del campo.**
Qué pasa: `public string Area { get { return _area; } }` con `int _area` da `error CS0029: Cannot implicitly convert type 'int' to 'string'`.
Por qué: el `get` debe devolver el tipo de la propiedad.
Arreglo: haz coincidir los tipos.

**5. Escribir paréntesis en los accesores.**
Qué pasa: `get() { ... }` da errores de sintaxis.
Por qué: `get` y `set` no son métodos normales y no llevan paréntesis.
Arreglo: `get { ... }` o `get => ...`.

**6. Propiedades de solo escritura.**
Qué pasa: compila (`public string Clave { set { ... } }`), pero no se puede leer, lo que confunde y rompe herramientas como la serialización.
Por qué: rara vez tiene sentido un dato que se escribe y nunca se lee.
Arreglo: usa un método (`EstablecerClave(...)`).

-----

## Según la versión de C#

* **C# 3:** propiedades autoimplementadas (`{ get; set; }`).
* **C# 6:** inicializadores de propiedades (`{ get; set; } = valor`), propiedades autoimplementadas de solo lectura (`{ get; }`) y propiedades con cuerpo de expresión (`=> ...`).
* **C# 7:** accesores con cuerpo de expresión (`get => ...; set => ...;`).
* **C# 9:** accesor `init`.
* **C# 11:** modificador `required`.
* **C# 13:** propiedades `partial` (para generadores de código).
* **C# 14:** palabra clave `field` para acceder al campo de respaldo generado.

-----

## Cuándo sí y cuándo no

**Usa una propiedad autoimplementada cuando:**

* El dato no tiene reglas. Es el caso más común.

**Usa una propiedad con lógica cuando:**

* Hay que validar o normalizar el valor al asignarlo.

**Usa una propiedad calculada cuando:**

* El valor se deriva de otros datos y no quieres que quede desactualizado.

**Usa un método en lugar de una propiedad cuando:**

* La operación es costosa (consulta una base de datos, hace un cálculo pesado) o tiene efectos secundarios. Quien lee `objeto.Algo` espera algo rápido y sin sorpresas.
* La operación necesita parámetros.

-----

## Resumen en 5 líneas

1. Una propiedad se usa como un campo, pero ejecuta `get` al leer y `set` al escribir (`value` es lo asignado).
2. `{ get; set; }` es una propiedad autoimplementada: el compilador crea el campo de respaldo.
3. Valida en el `set`; ante valores inválidos, normalmente lanza `ArgumentOutOfRangeException`.
4. `{ get; }`, `{ get; private set; }`, `{ get; init; }` y `required` controlan quién y cuándo puede escribir.
5. `public double Area => Ancho * Alto;` es una propiedad calculada de solo lectura.

-----

## Para profundizar

<details>
<summary>Qué genera el compilador</summary>

Esta propiedad:

```csharp
public int Area { get; set; }
```

se compila, aproximadamente, como:

```csharp
private int <Area>k__BackingField;
public int get_Area() => <Area>k__BackingField;
public void set_Area(int value) => <Area>k__BackingField = value;
```

Por eso cambiar un campo público por una propiedad **rompe la compatibilidad binaria** de una librería: el código compilado contra ella accedía a un campo y ahora tendría que llamar a métodos. Es otra razón para empezar con propiedades desde el principio.

</details>

<details>
<summary>Propiedades e INotifyPropertyChanged</summary>

En aplicaciones de escritorio (WPF, MAUI), la interfaz gráfica necesita saber cuándo cambia un dato. Las propiedades con lógica en el `set` permiten avisar:

```csharp
public string Nombre
{
    get;
    set
    {
        if (field == value) return;
        field = value;
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Nombre)));
    }
}
```

Es uno de los usos más comunes de los setters con lógica, y una de las motivaciones de la palabra clave `field`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una propiedad expone un dato con la sintaxis de un campo, pero con accesores `get` y `set` que permiten validar o calcular. Las propiedades autoimplementadas (`{ get; set; }`) generan el campo automáticamente. Se puede restringir la escritura con `private set`, `init` o quitando el `set`.

### Respuesta ampliada (semi-senior)

Una propiedad se compila a métodos `get_X`/`set_X` y, si es autoimplementada, a un campo de respaldo generado. Se prefieren a los campos públicos porque permiten agregar lógica sin romper la API (los campos y las propiedades no son compatibles a nivel binario) y porque los serializadores y los ORMs trabajan con propiedades. `init` (C# 9) y `required` (C# 11) permiten objetos inmutables construidos con inicializadores; `field` (C# 14) habilita lógica en el setter sin declarar el campo. Por convención, una propiedad debe ser barata y sin efectos secundarios observables; si no, corresponde un método.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre campo y propiedad?**
Un campo es una variable de la clase. Una propiedad es un par de métodos con sintaxis de campo, que puede validar, calcular y controlar el acceso.

**2. ¿Diferencia entre `{ get; }`, `{ get; private set; }` e `{ get; init; }`?**
`{ get; }` solo se asigna en el constructor. `private set` permite modificarla desde cualquier método de la clase. `init` permite asignarla en el inicializador del objeto, y después queda fija.

**3. ¿Cuándo usar un método en lugar de una propiedad?**
Cuando la operación es costosa, tiene efectos secundarios, puede fallar con frecuencia o necesita parámetros.

-----

## Práctica

**Ejercicio 1.** Crea una clase `Alumno` con:

* `Nombre` (obligatorio al crear el objeto y que no cambie después).
* `Nota` (entre 0 y 20; lanza una excepción si no).
* `Aprobado` (calculada: `true` si la nota es 13 o más).

<details>
<summary>Solución</summary>

```csharp
var a = new Alumno { Nombre = "Luis", Nota = 15 };
Console.WriteLine($"{a.Nombre}: {a.Nota} → {(a.Aprobado ? "Aprobado" : "Desaprobado")}");

class Alumno
{
    private int _nota;

    public required string Nombre { get; init; }

    public int Nota
    {
        get => _nota;
        set
        {
            if (value is < 0 or > 20)
                throw new ArgumentOutOfRangeException(nameof(value), "La nota debe estar entre 0 y 20.");
            _nota = value;
        }
    }

    public bool Aprobado => Nota >= 13;
}
```

</details>

**Ejercicio 2.** ¿Qué tiene de malo esta propiedad? Corrígela.

```csharp
class Temperatura
{
    public double Celsius
    {
        get { return Celsius; }
        set { Celsius = value; }
    }
}
```

<details>
<summary>Solución</summary>

Se llama a sí misma: leer `Celsius` ejecuta el `get`, que lee `Celsius`, que ejecuta el `get`... hasta un `StackOverflowException`. Hay que usar un campo de respaldo, o simplemente una propiedad autoimplementada:

```csharp
class Temperatura
{
    public double Celsius { get; set; }
    public double Fahrenheit => Celsius * 9 / 5 + 32;   // de paso, una propiedad calculada
}
```

</details>

-----

## Siguiente lección

[Constructores y this](04-Constructores%20y%20this.md)
