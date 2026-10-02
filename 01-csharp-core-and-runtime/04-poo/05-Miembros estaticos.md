# Miembros estáticos

## En una frase

Un miembro `static` pertenece a la **clase**, no a cada objeto: hay **una sola copia** compartida, se usa con el nombre de la clase (`Math.PI`, `Console.WriteLine`) y no tiene acceso a los datos de ninguna instancia.

-----

## Antes de empezar

Conviene que ya sepas:

* Crear clases con campos, propiedades, métodos y constructores, de [Constructores y this](04-Constructores%20y%20this.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Miembro de instancia:** pertenece a cada objeto; cada uno tiene su propia copia.
* **Miembro estático (`static`):** pertenece a la clase; existe una sola copia compartida.
* **Constructor estático:** se ejecuta una sola vez, antes del primer uso de la clase.
* **Clase estática:** clase que solo tiene miembros estáticos y no se puede instanciar.
* **`static readonly`:** campo estático que se asigna una vez y después no cambia.
* **Punto de entrada (`Main`):** el método donde empieza el programa.
* **Estado global:** datos accesibles y modificables desde cualquier parte del programa.

-----

## El problema

Tienes una clase `Bosque`, y cada bosque tiene su nombre y su área. Ahora necesitas:

* **Contar cuántos bosques se crearon** en total.
* Guardar **la definición de "bosque"**, que es la misma para todos.
* Una función que **convierta hectáreas a km²**, que no depende de ningún bosque en particular.

¿Dónde pones esos datos? Si el contador es un campo normal, cada bosque tiene su propio contador, y ninguno sabe cuántos hay en total. La definición repetida en cada objeto es un desperdicio. Y para convertir unidades tendrías que crear un bosque "falso" solo para llamar al método.

Esos datos y comportamientos pertenecen **al concepto** `Bosque`, no a un bosque concreto.

-----

## Cómo funciona

### Campos y propiedades estáticas

```csharp
class Bosque
{
    public string Nombre { get; }                       // de instancia: uno por objeto
    public static int Cantidad { get; private set; }    // estático: uno para toda la clase
    public static string Definicion { get; } = "Área extensa cubierta de árboles.";

    public Bosque(string nombre)
    {
        Nombre = nombre;
        Cantidad++;                                     // todos los bosques incrementan el MISMO contador
    }
}
```

```csharp
var a = new Bosque("Amazonas");
var c = new Bosque("Congo");

Console.WriteLine(Bosque.Cantidad);     // 2  → se accede con el nombre de la CLASE
Console.WriteLine(Bosque.Definicion);
Console.WriteLine(a.Nombre);            // Amazonas → con el objeto
```

```text
        clase Bosque
   ┌───────────────────────────┐
   │ Cantidad = 2   (static)   │  ← una sola copia
   │ Definicion     (static)   │
   └───────────────────────────┘
        ▲               ▲
  objeto a          objeto c
  Nombre="Amazonas" Nombre="Congo"   ← una copia por objeto
```

* `static` va después del modificador de acceso: `public static`.
* Fuera de la clase, los miembros estáticos se usan con **el nombre de la clase**. Usarlos a través de un objeto es un error:

```csharp
Console.WriteLine(a.Cantidad);   // error CS0176: no se puede acceder al miembro estático con una referencia de instancia
```

### Métodos estáticos

```csharp
class Bosque
{
    public string Nombre { get; }
    public double Hectareas { get; }

    public Bosque(string nombre, double hectareas) => (Nombre, Hectareas) = (nombre, hectareas);

    public static double HectareasAKm2(double hectareas) => hectareas / 100;   // no depende de ningún bosque

    public double AreaKm2() => HectareasAKm2(Hectareas);                       // de instancia: usa SUS hectáreas
}

Console.WriteLine(Bosque.HectareasAKm2(500));           // 5, sin crear ningún bosque
Console.WriteLine(new Bosque("Congo", 300).AreaKm2());  // 3
```

Un método estático **no tiene `this`**, así que no puede usar directamente miembros de instancia:

```csharp
public static void Describir()
{
    Console.WriteLine(Nombre);   // error CS0120: se requiere una referencia de objeto para el miembro no estático 'Bosque.Nombre'
}
```

Tiene sentido: si llamas a `Bosque.Describir()`, ¿el nombre de qué bosque debería imprimir? Si necesita datos de un objeto, recíbelo como parámetro.

| Desde... | ¿Puede usar miembros de instancia? | ¿Puede usar miembros estáticos? |
| --- | --- | --- |
| Un método de instancia | Sí | Sí |
| Un método estático | No (salvo a través de un objeto) | Sí |

### Constantes y `static readonly`

```csharp
class Configuracion
{
    public const int MaxIntentos = 3;                                  // const: implícitamente estático
    public static readonly DateTime Inicio = DateTime.Now;              // se calcula al ejecutar
    public static readonly string[] Paises = { "Perú", "Chile" };       // cualquier tipo
}
```

| | `const` | `static readonly` |
| --- | --- | --- |
| Cuándo se conoce el valor | Al compilar | Al ejecutar (una vez) |
| Tipos permitidos | Numéricos, `bool`, `char`, `string`, `enum` | Cualquiera |
| Se puede usar `DateTime.Now`, `new ...` | No | Sí |

Una `const` ya es estática: escribir `static const` es un error (CS0504).

### Constructor estático

Inicializa los datos estáticos de la clase. Se ejecuta **una sola vez**, automáticamente, antes de que se cree el primer objeto o se use el primer miembro estático:

```csharp
class Catalogo
{
    public static Dictionary<string, decimal> Precios { get; }

    static Catalogo()                    // sin modificador de acceso y sin parámetros
    {
        Console.WriteLine("Cargando catálogo...");
        Precios = new Dictionary<string, decimal>
        {
            ["teclado"] = 150m,
            ["mouse"] = 60m
        };
    }
}

Console.WriteLine(Catalogo.Precios["mouse"]);   // "Cargando catálogo..." y luego 60
Console.WriteLine(Catalogo.Precios["teclado"]); // 150 (el constructor ya no se vuelve a ejecutar)
```

Reglas:

* No lleva modificador de acceso (CS0515) ni parámetros (CS0132).
* Solo puede haber uno por clase.
* No se puede llamar directamente; el runtime decide cuándo, garantizando que sea antes del primer uso.

### Clases estáticas

Si una clase **solo** tiene miembros estáticos y no tiene sentido crear objetos de ella, márcala `static`:

```csharp
static class Conversor
{
    public static double CelsiusAFahrenheit(double c) => c * 9 / 5 + 32;
    public static double KmAMillas(double km) => km * 0.621371;
}

Console.WriteLine(Conversor.CelsiusAFahrenheit(25));   // 77
var c = new Conversor();   // error CS0712: no se puede crear una instancia de la clase estática 'Conversor'
```

Una clase estática:

* No se puede instanciar ni heredar.
* Solo puede contener miembros estáticos (CS0708 si declaras uno de instancia).

Ya las usaste: **`Math`** y **`Console`** son clases estáticas. Por eso escribes `Math.Max(3, 7)` y `Console.WriteLine(...)` sin `new`.

### `using static`

Permite usar los miembros estáticos de una clase sin escribir su nombre:

```csharp
using static System.Math;

Console.WriteLine(Sqrt(16) + Max(3, 7) + PI);
```

Úsalo con moderación: pierdes de vista de dónde viene cada método.

### `Main`, parte por parte

Ahora puedes explicar cada palabra del punto de entrada clásico:

```csharp
class Program
{
    public static void Main(string[] args)
    {
    }
}
```

| Parte | Significado |
| --- | --- |
| `Main` | Nombre especial que el runtime busca como punto de entrada. |
| `static` | Se ejecuta sin crear un objeto `Program`: al arrancar, todavía no existe ninguno. |
| `void` | No devuelve nada. Puede ser `int` para devolver un código de salida al sistema operativo. |
| `string[] args` | Los argumentos de la línea de comandos: `dotnet run -- hola 42` → `args = ["hola", "42"]`. Es opcional. |
| `public` | **No es obligatorio**: `Main` funciona igual siendo `private`. Las plantillas antiguas lo ponían, pero el runtime no lo necesita. |

Formas válidas: `static void Main()`, `static int Main(string[] args)`, `static async Task Main()`, `static async Task<int> Main(string[] args)`.

Por eso, en programas con `Main` explícito, los métodos auxiliares de `Program` deben ser `static`: `Main` es estático y no tiene `this` sobre el cual llamar métodos de instancia.

-----

## Ejemplo completo

```csharp
var u1 = Usuario.Registrar("ana@mail.com");
var u2 = Usuario.Registrar("LUIS@mail.com");
var u3 = Usuario.Registrar("correo-invalido");

Console.WriteLine($"Usuarios registrados: {Usuario.Total}");
Console.WriteLine($"{u1?.Id} {u1?.Email}");
Console.WriteLine($"{u2?.Id} {u2?.Email}");
Console.WriteLine($"u3 es null: {u3 is null}");
Console.WriteLine(Validador.EsEmail("x@y.com"));

class Usuario
{
    private static int _ultimoId;                      // compartido: el último id asignado

    public static int Total { get; private set; }      // compartido: cuántos se crearon
    public static readonly DateTime Arranque = DateTime.Now;

    public int Id { get; }                             // de instancia
    public string Email { get; }

    private Usuario(string email)                      // constructor privado...
    {
        Id = ++_ultimoId;
        Email = email;
        Total++;
    }

    // ...y un método de fábrica estático que valida antes de crear
    public static Usuario? Registrar(string email)
    {
        if (!Validador.EsEmail(email)) return null;
        return new Usuario(email.Trim().ToLowerInvariant());
    }
}

static class Validador
{
    public static bool EsEmail(string texto) =>
        !string.IsNullOrWhiteSpace(texto) && texto.Contains('@') && texto.IndexOf('@') < texto.LastIndexOf('.');
}
```

Salida:

```text
Usuarios registrados: 2
1 ana@mail.com
2 luis@mail.com
u3 es null: True
True
```

Tres usos de `static`: un contador compartido (`Total`, `_ultimoId`), un método de fábrica (`Registrar`) y una clase utilitaria (`Validador`).

-----

## Errores comunes

**1. Usar un miembro de instancia desde un método estático.**
Qué pasa: `error CS0120: An object reference is required for the non-static field, method, or property 'Bosque.Nombre'`.
Por qué: un método estático no tiene `this`.
Arreglo: haz estático el miembro (si de verdad es compartido) o recibe el objeto como parámetro.

**2. Usar un miembro estático a través de un objeto.**
Qué pasa: `error CS0176: Member 'Bosque.Cantidad' cannot be accessed with an instance reference; qualify it with a type name instead`.
Por qué: el miembro no pertenece al objeto.
Arreglo: `Bosque.Cantidad`.

**3. Poner un modificador de acceso al constructor estático.**
Qué pasa: `error CS0515: 'Catalogo.Catalogo()': access modifiers are not allowed on static constructors`.
Por qué: nadie lo llama directamente; lo llama el runtime.
Arreglo: `static Catalogo() { }`.

**4. Instanciar una clase estática.**
Qué pasa: `error CS0712: Cannot create an instance of the static class 'Conversor'`.
Por qué: una clase estática no tiene constructores de instancia.
Arreglo: llama a sus miembros con el nombre de la clase.

**5. Declarar un miembro de instancia en una clase estática.**
Qué pasa: `error CS0708: 'Conversor.Factor': cannot declare instance members in a static class`.
Por qué: una clase estática nunca tiene objetos.
Arreglo: márcalo `static` o quita `static` de la clase.

**6. Usar estado estático mutable como "variable global".**
Qué pasa: no hay error de compilación, pero aparecen bugs difíciles de reproducir: una prueba deja datos que rompen otra, o dos hilos modifican el mismo contador a la vez.
Por qué: el estado estático vive durante toda la ejecución y lo comparte todo el programa.
Arreglo: limita lo estático a constantes, utilidades sin estado y métodos de fábrica. Si necesitas compartir un servicio, usa inyección de dependencias.

-----

## Según la versión de C#

* **C# 2:** clases estáticas.
* **C# 3:** métodos de extensión (métodos estáticos en clases estáticas que se usan como si fueran de instancia: `texto.MiMetodo()`).
* **C# 6:** `using static`.
* **C# 7.1:** `Main` asíncrono (`static async Task Main`).
* **C# 9:** top-level statements, que ocultan el `Main` estático.
* **C# 11:** miembros `static abstract` en interfaces.
* **C# 14:** bloques `extension`, que permiten declarar propiedades y miembros estáticos de extensión.

-----

## Cuándo sí y cuándo no

**Usa `static` cuando:**

* El dato es una constante o configuración compartida (`const`, `static readonly`).
* El método no usa el estado de ningún objeto: utilidades puras (`Math.Max`, conversores).
* Necesitas métodos de fábrica (`Usuario.Registrar`, `Guid.NewGuid`).

**No uses `static` cuando:**

* El dato pertenece a cada objeto (nombre, saldo, estado).
* Quieres poder reemplazar la implementación en las pruebas: los métodos estáticos no se pueden sustituir fácilmente; una interfaz inyectada sí.
* Es estado mutable compartido "por comodidad".

-----

## Resumen en 5 líneas

1. `static` = pertenece a la clase; hay una sola copia y se usa con el nombre de la clase.
2. Los métodos estáticos no tienen `this`: no pueden usar miembros de instancia directamente.
3. El constructor estático se ejecuta una vez, antes del primer uso, sin modificadores ni parámetros.
4. Una clase `static` solo tiene miembros estáticos y no se instancia (`Math`, `Console`).
5. `Main` es `static` porque se ejecuta antes de que exista cualquier objeto.

-----

## Para profundizar

<details>
<summary>Métodos de extensión</summary>

Un método estático en una clase estática, con `this` delante del primer parámetro, se puede llamar como si fuera un método del tipo extendido:

```csharp
static class StringExtensions
{
    public static bool EsPalindromo(this string texto)
    {
        string limpio = texto.Replace(" ", "").ToLowerInvariant();
        return limpio.SequenceEqual(limpio.Reverse());
    }
}

Console.WriteLine("Anita lava la tina".EsPalindromo());   // True
```

Todo LINQ (`Where`, `Select`...) está construido con métodos de extensión sobre `IEnumerable<T>`.

</details>

<details>
<summary>Estado estático e hilos</summary>

`Total++` no es una operación atómica: lee, suma y escribe. Si dos hilos crean usuarios a la vez, pueden leer el mismo valor y perder un incremento. Para contadores compartidos entre hilos se usa `Interlocked.Increment(ref _total)`. Es otra razón para evitar el estado estático mutable.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un miembro `static` pertenece a la clase y no a los objetos: hay una sola copia compartida y se accede con el nombre de la clase. Los métodos estáticos no pueden usar miembros de instancia porque no tienen `this`. Una clase estática solo tiene miembros estáticos y no se puede instanciar, como `Math` o `Console`.

### Respuesta ampliada (semi-senior)

Los campos estáticos se almacenan una vez por tipo (por tipo genérico cerrado, en el caso de los genéricos) y viven durante toda la vida del proceso. El constructor estático es *thread-safe* y lo ejecuta el runtime antes del primer acceso; sin constructor estático explícito, el tipo se marca `beforefieldinit` y la inicialización puede ocurrir antes. Las clases estáticas son `abstract sealed` en IL. Lo estático es adecuado para funciones puras, constantes y fábricas, pero el estado estático mutable introduce acoplamiento global, problemas de concurrencia y dificulta las pruebas, por lo que en aplicaciones se prefiere registrar servicios como *singleton* en el contenedor de inyección de dependencias.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `const` y `static readonly`?**
`const` se resuelve al compilar, solo admite tipos primitivos y `string`, y se copia en el código que la usa. `static readonly` se asigna en ejecución una vez y admite cualquier tipo.

**2. ¿Por qué `Main` es estático?**
Porque cuando arranca el programa no existe ningún objeto sobre el cual llamar un método de instancia.

**3. ¿Diferencia entre una clase estática y un Singleton?**
La clase estática no tiene instancia, no implementa interfaces ni se puede pasar como parámetro. Un Singleton es un objeto único: puede implementar interfaces, inyectarse y reemplazarse en pruebas.

-----

## Práctica

**Ejercicio 1.** Agrega a una clase `Ticket` un número correlativo automático (1, 2, 3...) usando un campo estático, y una propiedad estática `Emitidos` con la cantidad total. Crea tres tickets y muestra sus números y el total.

<details>
<summary>Solución</summary>

```csharp
var t1 = new Ticket("Soporte");
var t2 = new Ticket("Ventas");
var t3 = new Ticket("Soporte");

Console.WriteLine($"{t1.Numero} {t2.Numero} {t3.Numero}");   // 1 2 3
Console.WriteLine($"Emitidos: {Ticket.Emitidos}");          // Emitidos: 3

class Ticket
{
    public static int Emitidos { get; private set; }

    public int Numero { get; }
    public string Area { get; }

    public Ticket(string area)
    {
        Emitidos++;
        Numero = Emitidos;
        Area = area;
    }
}
```

</details>

**Ejercicio 2.** Este código no compila. Explica por qué y propón dos soluciones distintas.

```csharp
class Calculadora
{
    private double _iva = 0.18;

    public static double ConIva(double monto) => monto * (1 + _iva);
}
```

<details>
<summary>Solución</summary>

`ConIva` es estático y `_iva` es de instancia: `error CS0120`. Soluciones:

1. Si el IVA es fijo para todos, hazlo constante: `private const double Iva = 0.18;`.
2. Si cada calculadora puede tener su propio IVA, quita `static` del método y llámalo sobre un objeto: `new Calculadora().ConIva(100)`.

</details>

-----

## Siguiente lección

[Herencia](06-Herencia.md)
