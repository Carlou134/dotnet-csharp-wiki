# Interfaces

## En una frase

Una interfaz es un **contrato**: declara qué miembros (métodos, propiedades, eventos) debe tener una clase, sin decir cómo funcionan; una clase puede implementar **muchas** interfaces, y cualquier código puede trabajar con "algo que cumple el contrato" sin conocer la clase concreta.

-----

## Antes de empezar

Conviene que ya sepas:

* Herencia, `override` y clases abstractas, de [Virtual, override y clases abstractas](07-Virtual%20override%20y%20clases%20abstractas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Interfaz:** tipo que declara un conjunto de miembros que una clase se compromete a implementar.
* **Implementar una interfaz:** que una clase defina todos los miembros que la interfaz declara.
* **Contrato:** el conjunto de operaciones que una interfaz garantiza.
* **Implementación explícita:** implementar un miembro de forma que solo sea visible a través de la interfaz.
* **Método por defecto (*default interface method*):** miembro de interfaz con implementación (C# 8).
* **Inyección de dependencias:** pasarle a una clase los objetos que necesita (normalmente como interfaces) en lugar de que los cree ella.

-----

## El problema

C# es "seguro en cuanto a tipos": el compilador detecta muy temprano errores como usar un `int` donde va un `string`, mucho antes de que lleguen a producción. Pero ¿qué pasa con tus propias clases?

La policía de tránsito necesita que **todo** vehículo que circule tenga patente, velocidad, cantidad de ruedas y bocina, para poder controlarlos. Los fabricantes construyen sedanes, camiones, motos y, mañana, quién sabe qué. A la policía **no le importa cómo** funciona cada uno por dentro; solo necesita la garantía de que todos tienen esos miembros.

* Una clase base `Vehiculo` serviría, pero obligaría a todos a heredar de ella, y una clase solo puede heredar de **una**. ¿Y si una moto ya hereda de `VehiculoDeDosRuedas`?
* Sin ninguna relación, el compilador no puede verificar que un `Tractor` tenga patente hasta que el programa falla.

Lo que se necesita es un **contrato verificable por el compilador**, independiente de la jerarquía de herencia.

-----

## Cómo funciona

### Declarar una interfaz

```csharp
interface IAutomovil
{
    string Patente { get; }
    double Velocidad { get; }
    int Ruedas { get; }
    void TocarBocina();
}
```

* Se declara con `interface` y, por convención, el nombre empieza con **`I`** (`IAutomovil`, `IComparable`). No es obligatorio para el compilador, pero en .NET todo el mundo lo respeta.
* Los miembros **no tienen cuerpo**: solo la firma.
* Los miembros son **públicos implícitamente**: no hace falta escribir `public`.
* Puede declarar métodos, propiedades, eventos e indexadores. **No** puede tener campos de instancia ni constructores.

### Implementar una interfaz

```csharp
class Sedan : IAutomovil
{
    public string Patente { get; }
    public double Velocidad { get; private set; }
    public int Ruedas => 4;

    public Sedan(string patente) => Patente = patente;

    public void TocarBocina() => Console.WriteLine("¡Bip bip!");

    public void Acelerar() => Velocidad += 10;   // miembro extra: la interfaz no lo prohíbe
}
```

* Se usa la misma sintaxis `:` que en la herencia.
* La clase debe implementar **todos** los miembros, y como `public`. Si falta alguno:

```text
error CS0535: 'Sedan' does not implement interface member 'IAutomovil.TocarBocina()'
```

* La interfaz dice lo que la clase **debe** tener, no lo que **no puede** tener: `Sedan` agrega `Acelerar()` y su constructor sin problema.
* Fíjate en `Velocidad`: la interfaz pide solo `get`, y la clase la implementa con `{ get; private set; }`. Es válido: cumple el contrato (se puede leer) y la clase decide cómo escribirla.

### Varias clases, un mismo contrato

```csharp
class Camion : IAutomovil
{
    public string Patente { get; }
    public double Velocidad { get; private set; }
    public double PesoKg { get; }
    public int Ruedas => PesoKg > 10_000 ? 12 : 8;    // implementación distinta

    public Camion(string patente, double pesoKg)
    {
        Patente = patente;
        PesoKg = pesoKg;
    }

    public void TocarBocina() => Console.WriteLine("¡HOOONK!");
}
```

Ahora la policía trabaja con el **contrato**, no con las clases:

```csharp
static void Controlar(IAutomovil auto)
{
    Console.WriteLine($"{auto.Patente}: {auto.Ruedas} ruedas a {auto.Velocidad} km/h");
    auto.TocarBocina();
}

Controlar(new Sedan("ABC-123"));
Controlar(new Camion("XYZ-999", 15_000));
```

`Controlar` no conoce `Sedan` ni `Camion`. Mañana aparece `Moto : IAutomovil` y `Controlar` funciona sin cambiar una línea.

### La interfaz como tipo

Una interfaz se puede usar como tipo de variables, parámetros, retornos y colecciones, pero **no se puede instanciar**:

```csharp
IAutomovil a = new Sedan("ABC-123");     // ✅ la variable es del tipo interfaz
var flota = new List<IAutomovil> { new Sedan("A-1"), new Camion("B-2", 8000) };

IAutomovil x = new IAutomovil();          // ❌ error CS0144: no se puede crear una instancia del tipo abstracto o interfaz
```

A través de una variable `IAutomovil` solo se ven los miembros de la interfaz: `a.Acelerar()` no compila, aunque el objeto sea un `Sedan`. Para eso hace falta un *cast* (ver [Polimorfismo y casting](09-Polimorfismo%20y%20casting.md)).

### Implementar varias interfaces

A diferencia de la herencia, una clase puede implementar **todas las interfaces que necesite**, y además heredar de una clase base:

```csharp
interface IAsegurable
{
    decimal CalcularPrima();
}

interface IRastreable
{
    (double Lat, double Lon) Ubicacion { get; }
}

class Taxi : Vehiculo, IAutomovil, IAsegurable, IRastreable   // la clase base, siempre primero
{
    // ... implementa los miembros de las tres interfaces
}
```

Las interfaces modelan **capacidades** ("se puede asegurar", "se puede rastrear") que clases sin relación entre sí pueden compartir: un `Taxi` y un `Celular` pueden ser ambos `IRastreable`.

### Interfaces de .NET que ya usas

```csharp
IList<int> numeros = new List<int> { 10, 20, 30 };   // List<T> implementa IList<T>
IEnumerable<int> recorrible = numeros;                 // ...e IEnumerable<T>: por eso funciona foreach

foreach (int n in recorrible)
{
    Console.WriteLine(n);
}
```

| Interfaz | Contrato | Lo implementa |
| --- | --- | --- |
| `IEnumerable<T>` | Se puede recorrer con `foreach` | Arrays, `List<T>`, `string`... |
| `IComparable<T>` | Se puede comparar para ordenar | `int`, `string`, `DateTime`... |
| `IDisposable` | Libera recursos con `Dispose()` (y `using`) | Archivos, conexiones... |
| `IEquatable<T>` | Igualdad por valor fuertemente tipada | Tipos primitivos, records |

Implementar `IComparable<T>` en tu clase hace que `Array.Sort` y `List.Sort` sepan ordenarla. Ese es el poder de los contratos.

### Implementación explícita

Si dos interfaces declaran un miembro con el mismo nombre, o quieres que un miembro solo se vea a través de la interfaz:

```csharp
interface IImprimible { void Mostrar(); }
interface IPantalla { void Mostrar(); }

class Documento : IImprimible, IPantalla
{
    void IImprimible.Mostrar() => Console.WriteLine("Enviando a la impresora");
    void IPantalla.Mostrar() => Console.WriteLine("Mostrando en pantalla");
}

var d = new Documento();
((IImprimible)d).Mostrar();   // Enviando a la impresora
((IPantalla)d).Mostrar();     // Mostrando en pantalla
d.Mostrar();                  // error CS1061: 'Documento' no contiene una definición para 'Mostrar'
```

Los miembros explícitos no llevan modificador de acceso y solo se pueden usar a través de una referencia del tipo de la interfaz.

### Métodos por defecto (C# 8)

Desde C# 8, una interfaz puede incluir miembros **con implementación**. El objetivo principal es poder **agregar** un miembro a una interfaz ya publicada sin romper todas las clases que la implementan:

```csharp
interface ISaludo
{
    string Nombre { get; }
    string Saludar() => $"Hola, soy {Nombre}";   // implementación por defecto
}

class Persona : ISaludo
{
    public string Nombre { get; init; } = "";
}

ISaludo s = new Persona { Nombre = "Ana" };
Console.WriteLine(s.Saludar());                // Hola, soy Ana

var p = new Persona { Nombre = "Luis" };
p.Saludar();   // error CS1061: el método por defecto NO forma parte de la clase
```

El método por defecto solo es accesible **a través de la interfaz**, no de la clase. Es una diferencia importante con las clases abstractas.

### Interfaz frente a clase abstracta

| Característica | Interfaz | Clase abstracta |
| --- | --- | --- |
| ¿Cuántas puede usar una clase? | **Muchas** | **Una** sola |
| ¿Se puede instanciar? | No | No |
| Implementación de miembros | No (salvo métodos por defecto, C# 8+) | Sí, la que quieras |
| Campos de instancia y estado | No | Sí |
| Constructores | No | Sí |
| Modificadores de acceso en miembros | Públicos por defecto | Los que quieras (`protected`, etc.) |
| Expresa | Una **capacidad** o contrato: "puede hacer" | Una **identidad** común: "es un" |
| Uso típico | Desacoplar componentes, inyección de dependencias, patrones de diseño | Compartir código y estado entre clases muy relacionadas (Template Method) |

¿Y el rendimiento? A veces se dice que "las interfaces son más rápidas que las clases abstractas". **No es cierto**: una llamada a través de una interfaz suele ser apenas igual o un poco más costosa que una llamada virtual, y en ambos casos la diferencia es despreciable. La elección es de **diseño**, no de rendimiento.

-----

## Ejemplo completo

Un sistema de notificaciones desacoplado: el servicio solo conoce la interfaz.

```csharp
var servicio = new ServicioPedidos(new INotificador[]
{
    new NotificadorEmail("ventas@tienda.com"),
    new NotificadorSms("+51 999 888 777"),
    new NotificadorConsola()
});

servicio.ConfirmarPedido(1042, "Ana");

interface INotificador
{
    string Canal { get; }
    bool Enviar(string destinatario, string mensaje);
}

class NotificadorEmail : INotificador
{
    private readonly string _remitente;
    public NotificadorEmail(string remitente) => _remitente = remitente;

    public string Canal => "Email";

    public bool Enviar(string destinatario, string mensaje)
    {
        Console.WriteLine($"[Email de {_remitente} a {destinatario}] {mensaje}");
        return true;
    }
}

class NotificadorSms : INotificador
{
    private readonly string _numeroOrigen;
    public NotificadorSms(string numeroOrigen) => _numeroOrigen = numeroOrigen;

    public string Canal => "SMS";

    public bool Enviar(string destinatario, string mensaje)
    {
        string corto = mensaje.Length > 40 ? mensaje[..37] + "..." : mensaje;
        Console.WriteLine($"[SMS desde {_numeroOrigen} a {destinatario}] {corto}");
        return true;
    }
}

class NotificadorConsola : INotificador
{
    public string Canal => "Consola";
    public bool Enviar(string destinatario, string mensaje)
    {
        Console.WriteLine($"[Debug] {destinatario}: {mensaje}");
        return true;
    }
}

class ServicioPedidos
{
    private readonly INotificador[] _notificadores;

    // Las dependencias llegan desde afuera, como interfaces
    public ServicioPedidos(INotificador[] notificadores) => _notificadores = notificadores;

    public void ConfirmarPedido(int numero, string cliente)
    {
        string mensaje = $"Hola {cliente}, tu pedido #{numero} fue confirmado y será enviado pronto.";
        foreach (INotificador n in _notificadores)
        {
            bool ok = n.Enviar(cliente, mensaje);
            Console.WriteLine($"  → {n.Canal}: {(ok ? "enviado" : "falló")}");
        }
    }
}
```

Salida:

```text
[Email de ventas@tienda.com a Ana] Hola Ana, tu pedido #1042 fue confirmado y será enviado pronto.
  → Email: enviado
[SMS desde +51 999 888 777 a Ana] Hola Ana, tu pedido #1042 fue confirm...
  → SMS: enviado
[Debug] Ana: Hola Ana, tu pedido #1042 fue confirmado y será enviado pronto.
  → Consola: enviado
```

`ServicioPedidos` no sabe nada de emails ni de SMS. Agregar WhatsApp es crear una clase nueva que implemente `INotificador`, sin tocar el servicio. Y en las pruebas puedes pasarle un notificador falso que no envía nada. Esta idea es la base de la **inyección de dependencias** que usa ASP.NET Core.

-----

## Errores comunes

**1. No implementar todos los miembros.**
Qué pasa: `error CS0535: 'Sedan' does not implement interface member 'IAutomovil.TocarBocina()'`.
Por qué: implementar una interfaz es una promesa completa.
Arreglo: implementa el miembro (Ctrl + . → "Implementar interfaz" lo genera por ti).

**2. Implementar un miembro sin `public`.**
Qué pasa: `error CS0737: 'Sedan' does not implement interface member 'IAutomovil.TocarBocina()'. 'Sedan.TocarBocina()' cannot implement an interface member because it is not public.`
Por qué: los miembros de una interfaz son públicos; la implementación implícita también debe serlo.
Arreglo: agrega `public` (o usa implementación explícita).

**3. Instanciar una interfaz.**
Qué pasa: `error CS0144: Cannot create an instance of the abstract type or interface 'IAutomovil'`.
Por qué: una interfaz no tiene implementación propia.
Arreglo: instancia una clase que la implemente.

**4. Declarar un campo de instancia en una interfaz.**
Qué pasa: `error CS0525: Interfaces cannot contain instance fields`.
Por qué: las interfaces no tienen estado.
Arreglo: declara una propiedad: `string Patente { get; }`.

**5. Llamar a un miembro de la clase desde una variable de tipo interfaz.**
Qué pasa: `error CS1061: 'IAutomovil' does not contain a definition for 'Acelerar'`.
Por qué: la variable solo "ve" el contrato de la interfaz.
Arreglo: agrega el miembro a la interfaz (si forma parte del contrato) o haz un cast con `is` (ver la próxima lección).

**6. Llamar a un método por defecto desde la clase.**
Qué pasa: `error CS1061` al usar `persona.Saludar()`.
Por qué: los métodos por defecto pertenecen a la interfaz, no a la clase.
Arreglo: usa una variable del tipo de la interfaz, o implementa el método en la clase.

-----

## Según la versión de C#

* **C# 1:** interfaces con métodos, propiedades, eventos e indexadores sin implementación.
* **C# 8:** métodos con implementación por defecto, miembros estáticos y modificadores de acceso en interfaces.
* **C# 11:** miembros `static abstract` y `static virtual` en interfaces, base del *generic math* (`INumber<T>`).
* **C# 13:** los `ref struct` pueden implementar interfaces.

Si lees "las interfaces no pueden tener implementación", es una regla de antes de C# 8.

-----

## Cuándo sí y cuándo no

**Usa una interfaz cuando:**

* Quieres que el código dependa de **qué** se hace y no de **quién** lo hace (servicios, repositorios, notificadores).
* Clases sin relación de herencia deben compartir una capacidad.
* Necesitas poder reemplazar una implementación (en pruebas, o según la configuración).

**Usa una clase abstracta cuando:**

* Las clases comparten estado y código, además del contrato, y la relación es "es un".

**Evita:**

* Interfaces gigantes con muchos miembros: divídelas por responsabilidad (principio de segregación de interfaces, la I de SOLID). `IRepositorio` con 25 métodos obliga a cada implementación a escribir los 25.
* Crear una interfaz para cada clase "por costumbre", cuando nunca habrá una segunda implementación ni una prueba que la reemplace.

-----

## Resumen en 5 líneas

1. Una interfaz declara miembros sin implementación; por convención su nombre empieza con `I`.
2. `class X : IContrato` obliga a implementar todos los miembros como `public` (CS0535).
3. Una clase hereda de una sola clase, pero implementa todas las interfaces que quiera.
4. Las interfaces se usan como tipo (`IAutomovil a = new Sedan()`) pero no se instancian.
5. Interfaz = capacidad ("puede hacer"); clase abstracta = identidad con estado y código compartido ("es un").

-----

## Para profundizar

<details>
<summary>Miembros static abstract (C# 11)</summary>

Una interfaz puede exigir miembros **estáticos**. Así se pueden escribir métodos genéricos que llaman a operaciones del tipo, no de un objeto:

```csharp
interface IConValorCero<T>
{
    static abstract T Cero { get; }
}

readonly struct Dinero : IConValorCero<Dinero>
{
    public decimal Monto { get; init; }
    public static Dinero Cero => new() { Monto = 0 };
}

static T ObtenerCero<T>() where T : IConValorCero<T> => T.Cero;
```

Es lo que permite que .NET tenga `INumber<T>` y métodos genéricos que suman cualquier tipo numérico.

</details>

<details>
<summary>Interfaces e inyección de dependencias en ASP.NET Core</summary>

En ASP.NET Core se registra qué clase concreta corresponde a cada interfaz, y el framework la entrega donde se pida:

```csharp
builder.Services.AddScoped<INotificador, NotificadorEmail>();

public class PedidosController(INotificador notificador) { /* ... */ }
```

El controlador nunca hace `new NotificadorEmail()`: depende del contrato. Es la aplicación directa del principio de inversión de dependencias (la D de SOLID).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una interfaz define un contrato: los métodos y propiedades que una clase debe tener, sin implementarlos. Una clase puede implementar varias interfaces, pero heredar de una sola clase. Se usan para que el código dependa de lo que un objeto sabe hacer y no de su clase concreta, por ejemplo en la inyección de dependencias.

### Respuesta ampliada (semi-senior)

Las interfaces definen contratos sin estado de instancia y permiten la "herencia múltiple" de tipo. Desde C# 8 admiten implementaciones por defecto (accesibles solo a través de la interfaz, pensadas para evolucionar APIs publicadas) y miembros estáticos; desde C# 11, miembros `static abstract`, base del *generic math*. La implementación explícita resuelve conflictos de nombres y oculta miembros de la API pública de la clase. Frente a una clase abstracta, una interfaz no aporta estado ni constructores, pero no consume la única herencia de clase disponible. Son la pieza central de la inversión de dependencias, la testabilidad con dobles de prueba y la mayoría de los patrones de diseño (Strategy, Repository, Decorator).

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre una clase y una interfaz?**
Una clase define datos y comportamiento implementado, y se puede instanciar. Una interfaz solo define qué miembros deben existir; no se instancia y una clase puede implementar muchas.

**2. ¿Diferencia entre interfaz y clase abstracta?**
La clase abstracta puede tener estado, constructores y código compartido, pero solo se hereda de una. La interfaz es un contrato sin estado y se pueden implementar muchas. Interfaz para capacidades; clase abstracta para una base común con lógica.

**3. ¿Una interfaz puede tener propiedades?**
Sí, se declaran sin implementación: `string Nombre { get; }`. Lo que no puede tener son campos de instancia.

**4. ¿Una interfaz puede heredar de otra?**
Sí, y de varias: `interface IVehiculoElectrico : IAutomovil, IRecargable { }`.

-----

## Práctica

**Ejercicio 1.** Define una interfaz `IFigura` con `double Area()` y `string Nombre { get; }`. Impleméntala en `Circulo` y `Triangulo` (base y altura). Escribe un método que reciba un `IFigura[]` y devuelva la figura de mayor área.

<details>
<summary>Solución</summary>

```csharp
IFigura[] figuras = { new Circulo(1), new Triangulo(4, 3) };
IFigura mayor = MayorArea(figuras);
Console.WriteLine($"{mayor.Nombre}: {mayor.Area():F2}");   // Triangulo: 6.00

static IFigura MayorArea(IFigura[] figuras)
{
    IFigura mayor = figuras[0];
    foreach (var f in figuras)
    {
        if (f.Area() > mayor.Area()) mayor = f;
    }
    return mayor;
}

interface IFigura
{
    string Nombre { get; }
    double Area();
}

class Circulo : IFigura
{
    private readonly double _radio;
    public Circulo(double radio) => _radio = radio;

    public string Nombre => "Circulo";
    public double Area() => Math.PI * _radio * _radio;
}

class Triangulo : IFigura
{
    private readonly double _base, _altura;
    public Triangulo(double b, double h) => (_base, _altura) = (b, h);

    public string Nombre => "Triangulo";
    public double Area() => _base * _altura / 2;
}
```

</details>

**Ejercicio 2.** Haz que una clase `Producto` (con `Nombre` y `Precio`) implemente `IComparable<Producto>` para que `Array.Sort` ordene los productos por precio de menor a mayor.

<details>
<summary>Solución</summary>

```csharp
Producto[] productos =
{
    new("Monitor", 800m),
    new("Mouse", 60m),
    new("Teclado", 150m)
};

Array.Sort(productos);   // usa CompareTo

foreach (var p in productos)
    Console.WriteLine($"{p.Nombre}: {p.Precio}");
// Mouse: 60
// Teclado: 150
// Monitor: 800

class Producto : IComparable<Producto>
{
    public string Nombre { get; }
    public decimal Precio { get; }

    public Producto(string nombre, decimal precio) => (Nombre, Precio) = (nombre, precio);

    public int CompareTo(Producto? otro) => otro is null ? 1 : Precio.CompareTo(otro.Precio);
}
```

`CompareTo` devuelve un número negativo si este objeto va antes, 0 si son iguales y positivo si va después. `Array.Sort` no conoce `Producto`: solo sabe usar el contrato `IComparable<T>`.

</details>

-----

## Siguiente lección

[Polimorfismo y casting](09-Polimorfismo%20y%20casting.md)
