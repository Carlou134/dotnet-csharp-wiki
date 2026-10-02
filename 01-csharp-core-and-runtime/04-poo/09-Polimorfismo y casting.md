# Polimorfismo y casting

## En una frase

El polimorfismo permite tratar objetos de distintas clases a través de un tipo común (una clase base o una interfaz) y que cada uno responda **a su manera** a la misma llamada; el **casting** convierte la referencia entre esos tipos: hacia arriba (*upcasting*, implícito y seguro) o hacia abajo (*downcasting*, explícito y verificable con `is` y `as`).

-----

## Antes de empezar

Conviene que ya sepas:

* `virtual`, `override` y clases abstractas, de [Virtual, override y clases abstractas](07-Virtual%20override%20y%20clases%20abstractas.md).
* Usar interfaces como tipo, de [Interfaces](08-Interfaces.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Polimorfismo:** "muchas formas": la misma operación se comporta distinto según el tipo real del objeto.
* **Tipo estático (declarado):** el tipo de la variable, conocido al compilar.
* **Tipo dinámico (real):** el tipo del objeto al que apunta la variable, conocido al ejecutar.
* **Upcasting:** tratar un objeto derivado como su tipo base. Es implícito.
* **Downcasting:** tratar una referencia de tipo base como un tipo derivado. Es explícito y puede fallar.
* **`is`:** operador que comprueba si un objeto es compatible con un tipo.
* **`as`:** operador que intenta un cast y devuelve `null` si no puede.
* **Pattern matching:** comprobar el tipo (o la forma) de un valor y extraerlo en una variable en un solo paso.

-----

## El problema

Tienes `Perro`, `Gato` y `Pajaro`, todos derivados de `Animal`, y quieres que cada uno haga su sonido. Sin polimorfismo:

```csharp
foreach (object animal in animales)
{
    if (animal is Perro)
        Console.WriteLine("Guau");
    else if (animal is Gato)
        Console.WriteLine("Miau");
    else if (animal is Pajaro)
        Console.WriteLine("Pío");
    // ¿Y cuando agreguen Vaca? Hay que encontrar y modificar CADA if como este en todo el programa.
}
```

Este código tiene que conocer todas las clases, crece con cada animal nuevo y se repite en cada lugar donde se necesite el sonido. Es frágil y repetitivo.

Lo que quieres decir es simplemente: "**animal, haz tu sonido**", y que cada animal sepa cuál es el suyo.

-----

## Cómo funciona

### Polimorfismo con `virtual` y `override`

```csharp
class Animal
{
    public string Nombre { get; }
    public Animal(string nombre) => Nombre = nombre;

    public virtual string HacerSonido() => "...";
    public void Dormir() => Console.WriteLine($"{Nombre} duerme");   // no virtual
}

class Perro : Animal
{
    public Perro(string nombre) : base(nombre) { }
    public override string HacerSonido() => "Guau";
    public void TraerPelota() => Console.WriteLine($"{Nombre} trae la pelota");
}

class Gato : Animal
{
    public Gato(string nombre) : base(nombre) { }
    public override string HacerSonido() => "Miau";
}
```

```csharp
Animal[] animales = { new Perro("Firulais"), new Gato("Michi"), new Animal("Genérico") };

foreach (Animal a in animales)
{
    Console.WriteLine($"{a.Nombre}: {a.HacerSonido()}");
}
// Firulais: Guau
// Michi: Miau
// Genérico: ...
```

El bucle no tiene un solo `if`. Agregar `Vaca` es crear una clase con su `override`: **el bucle no cambia**. Esto es el principio abierto/cerrado (la O de SOLID): abierto a la extensión, cerrado a la modificación.

### Tipo estático y tipo dinámico

```csharp
Animal a = new Perro("Firulais");
//  ▲              ▲
//  │              └─ tipo DINÁMICO (real): Perro. Decide QUÉ implementación se ejecuta.
//  └─ tipo ESTÁTICO (declarado): Animal. Decide QUÉ miembros puedes usar.
```

Esta distinción explica todo el comportamiento:

```csharp
Animal a = new Perro("Firulais");

a.HacerSonido();     // "Guau": el método es virtual → decide el tipo dinámico
a.Dormir();          // existe en Animal → se puede llamar
a.TraerPelota();     // error CS1061: 'Animal' no contiene una definición para 'TraerPelota'
```

El compilador solo conoce el tipo estático (`Animal`) y no sabe que, en este caso, el objeto es un `Perro`. Por eso no te deja llamar a `TraerPelota()`.

### Upcasting: de derivado a base

```text
                 object
                   ▲
                 Animal          ▲ UPCASTING: implícito, siempre seguro
                ▲      ▲         │ (todo Perro es un Animal)
            Perro      Gato      │
                                 ▼ DOWNCASTING: explícito, puede fallar
                                   (no todo Animal es un Perro)

Animal a = new Perro();     variable "a" (tipo estático Animal) ──► objeto Perro en el heap (tipo dinámico)
                            ve: Nombre, HacerSonido(), Dormir()      tiene además: TraerPelota()
```

Asignar un objeto derivado a una variable del tipo base es un **upcasting**. Es **implícito**, porque siempre es seguro: todo `Perro` es un `Animal`.

```csharp
Perro p = new Perro("Firulais");
Animal a = p;              // upcasting implícito
IComparable c = "texto";   // también hacia interfaces
object o = p;              // todo es un object
```

A través de la referencia "subida" se pueden usar:

* Los miembros declarados en el tipo base.
* Y, si son virtuales, se ejecutan las versiones sobrescritas de la derivada.

### Downcasting: de base a derivado

Para volver a usar los miembros propios de `Perro`, necesitas un **downcasting**. Es **explícito** porque puede fallar: no todo `Animal` es un `Perro`.

```csharp
Animal a = new Perro("Firulais");
Perro p = (Perro)a;        // cast explícito: OK, el objeto real es un Perro
p.TraerPelota();

Animal g = new Gato("Michi");
Perro p2 = (Perro)g;       // compila, pero al ejecutar: System.InvalidCastException
```

El cast no transforma el objeto: solo cambia el tipo con el que lo ves. Si el objeto real no es de ese tipo, falla al ejecutar. Por eso, antes de un downcast, hay que **verificar**.

### `is`: comprobar el tipo

```csharp
Animal a = new Perro("Firulais");

if (a is Perro)
{
    Console.WriteLine("Es un perro");
}

Console.WriteLine(a is Animal);   // True: un Perro también es un Animal
Console.WriteLine(a is Gato);     // False
```

`is` devuelve `true` si el objeto es de ese tipo **o de uno derivado** (o si implementa esa interfaz).

### `is` con patrón: comprobar y convertir en un paso

Desde C# 7, `is` puede declarar una variable que ya tiene el tipo convertido:

```csharp
if (a is Perro perro)            // si es Perro, lo asigna a 'perro' (ya tipado como Perro)
{
    perro.TraerPelota();         // sin cast adicional
}

if (a is not Gato)               // negación (C# 9)
{
    Console.WriteLine("No es un gato");
}
```

Es la forma recomendada de hacer un downcast seguro.

### `as`: intentar el cast y obtener `null` si falla

```csharp
Animal a = new Gato("Michi");

Perro? p = a as Perro;    // no lanza excepción: devuelve null si no es un Perro
if (p != null)
{
    p.TraerPelota();
}
else
{
    Console.WriteLine("No era un perro");
}
```

* `as` solo funciona con tipos de referencia y tipos que aceptan `null` (con un `int`, da `error CS0077`).
* Hoy, `is Perro p` suele reemplazar a `as` + comprobación de `null`, porque es más corto y no deja una variable posiblemente nula fuera del `if`.

### Comparación de las tres formas

| Forma | Si el tipo no coincide | Úsala cuando... |
| --- | --- | --- |
| `(Perro)a` | Lanza `InvalidCastException` | Estás **seguro** del tipo y un error sería un bug |
| `a as Perro` | Devuelve `null` | Necesitas la referencia (o `null`) para usarla después |
| `a is Perro p` | Devuelve `false` | Quieres comprobar y usar en el mismo paso (la opción habitual) |

### Polimorfismo con interfaces

Funciona igual con interfaces: el tipo estático es la interfaz y cada clase implementa a su manera.

```csharp
IPago[] pagos = { new PagoTarjeta(), new PagoTransferencia(), new PagoEfectivo() };
foreach (IPago pago in pagos)
{
    Console.WriteLine($"{pago.GetType().Name}: {pago.Comision(200m):N2}");
}
// PagoTarjeta: 7.00
// PagoTransferencia: 1.50
// PagoEfectivo: 0.00

interface IPago
{
    decimal Comision(decimal monto);
}

class PagoTarjeta : IPago { public decimal Comision(decimal m) => m * 0.035m; }
class PagoTransferencia : IPago { public decimal Comision(decimal m) => 1.50m; }
class PagoEfectivo : IPago { public decimal Comision(decimal m) => 0m; }
```

### Las formas de polimorfismo en C#

| Tipo | Mecanismo | Se resuelve | Ejemplo |
| --- | --- | --- | --- |
| **De subtipos** (el "polimorfismo" por excelencia) | `virtual`/`override`, interfaces | Al ejecutar | `animal.HacerSonido()` |
| **Ad hoc** | Sobrecarga de métodos y operadores | Al compilar | `Sumar(int, int)` / `Sumar(double, double)` |
| **Paramétrico** | Genéricos | Al compilar (y el JIT especializa) | `List<T>` funciona con cualquier `T` |

Cuando en una entrevista te preguntan por "polimorfismo" en POO, casi siempre se refieren al primero.

-----

## Ejemplo completo

Un sistema de pagos en el que el procesamiento general es polimórfico y solo un caso especial usa casting:

```csharp
var pagos = new List<MedioDePago>
{
    new Tarjeta("4111-1111-1111-1234", 500m),
    new Billetera("ana@pay", 200m),
    new Tarjeta("5500-0000-0000-9876", 50m),
    new Efectivo(80m)
};

decimal totalComisiones = 0;

foreach (MedioDePago pago in pagos)
{
    // Polimorfismo: cada medio calcula y describe a su manera
    decimal comision = pago.CalcularComision();
    totalComisiones += comision;
    Console.WriteLine($"{pago.Describir(),-40} comisión: {comision,6:N2}");

    // Casting: solo las tarjetas tienen un comportamiento extra
    if (pago is Tarjeta tarjeta && tarjeta.Monto > 100m)
    {
        Console.WriteLine($"   ↳ Requiere validación 3D Secure para la tarjeta {tarjeta.Ultimos4}");
    }
}

Console.WriteLine($"Total de comisiones: {totalComisiones:N2}");

abstract class MedioDePago
{
    public decimal Monto { get; }
    protected MedioDePago(decimal monto) => Monto = monto;

    public abstract decimal CalcularComision();
    public virtual string Describir() => $"{GetType().Name} por {Monto:N2}";
}

class Tarjeta : MedioDePago
{
    private readonly string _numero;
    public Tarjeta(string numero, decimal monto) : base(monto) => _numero = numero;

    public string Ultimos4 => _numero[^4..];
    public override decimal CalcularComision() => Monto * 0.035m;
    public override string Describir() => $"Tarjeta ****{Ultimos4} por {Monto:N2}";
}

class Billetera : MedioDePago
{
    private readonly string _cuenta;
    public Billetera(string cuenta, decimal monto) : base(monto) => _cuenta = cuenta;

    public override decimal CalcularComision() => Math.Min(Monto * 0.02m, 3m);
    public override string Describir() => $"Billetera {_cuenta} por {Monto:N2}";
}

class Efectivo : MedioDePago
{
    public Efectivo(decimal monto) : base(monto) { }
    public override decimal CalcularComision() => 0m;
}
```

Salida:

```text
Tarjeta ****1234 por 500.00              comisión:  17.50
   ↳ Requiere validación 3D Secure para la tarjeta 1234
Billetera ana@pay por 200.00             comisión:   3.00
Tarjeta ****9876 por 50.00               comisión:   1.75
Efectivo por 80.00                       comisión:   0.00
Total de comisiones: 22.25
```

Lo general (comisión, descripción) se resuelve con polimorfismo. El `is Tarjeta tarjeta` queda para un comportamiento que **solo** tiene sentido en un tipo concreto. Si empezaras a tener `if (pago is X)` para cada tipo, sería la señal de mover ese comportamiento a un método virtual.

-----

## Errores comunes

**1. Llamar a un miembro de la derivada a través del tipo base.**
Qué pasa: `error CS1061: 'Animal' does not contain a definition for 'TraerPelota' and no accessible extension method 'TraerPelota' accepting a first argument of type 'Animal' could be found`.
Por qué: el compilador solo conoce el tipo estático.
Arreglo: `if (a is Perro p) p.TraerPelota();`.

**2. Downcast a un tipo incorrecto.**
Qué pasa: `System.InvalidCastException: Unable to cast object of type 'Gato' to type 'Perro'.`
Por qué: el objeto real no es de ese tipo.
Arreglo: comprueba con `is` antes, o usa `is Perro p`.

**3. Usar `as` con un tipo de valor.**
Qué pasa: `error CS0077: The as operator must be used with a reference type or nullable type ('int' is a non-nullable value type)`.
Por qué: `as` devuelve `null` si falla, y un `int` no puede ser `null`.
Arreglo: `if (obj is int n)` o `obj as int?`.

**4. Usar el resultado de `as` sin comprobar `null`.**
Qué pasa: `NullReferenceException` más adelante, lejos del lugar real del problema.
Por qué: `as` falla en silencio.
Arreglo: comprueba `null` inmediatamente o usa `is Tipo variable`.

**5. Olvidar `override` y esperar polimorfismo.**
Qué pasa: se ejecuta la versión de la base aunque el objeto sea derivado (con `warning CS0114`).
Por qué: sin `virtual`/`override` no hay despacho dinámico.
Arreglo: `virtual` en la base y `override` en la derivada.

**6. Cadenas de `if (x is A) ... else if (x is B) ...`.**
Qué pasa: compila, pero cada tipo nuevo obliga a modificar todas esas cadenas.
Por qué: es lógica que pertenece a cada clase.
Arreglo: mueve el comportamiento a un método virtual o a un miembro de interfaz.

-----

## Según la versión de C#

* **C# 1:** cast `(T)`, `is` y `as`.
* **C# 7:** patrones de tipo: `if (a is Perro p)` y `case Perro p:` en `switch`.
* **C# 8:** expresiones `switch` con patrones de tipo y de propiedades.
* **C# 9:** `is not`, patrones combinados (`is Perro or Gato`) y patrones de tipo sin variable en `switch` (`Perro => ...`).

```csharp
string Descripcion(Animal a) => a switch
{
    Perro { Nombre: "Firulais" } => "El perro famoso",
    Perro => "Un perro",
    Gato or Pajaro => "Un gato o un pájaro",
    _ => "Otro animal"
};
```

-----

## Cuándo sí y cuándo no

**Usa polimorfismo cuando:**

* Varios tipos comparten una operación que cada uno implementa a su manera. Es la opción por defecto.

**Usa downcasting (`is Tipo t`) cuando:**

* Un comportamiento solo existe en un tipo concreto y no tiene sentido en la base.
* Trabajas con datos de tipo `object` o heterogéneos que vienen de afuera.

**Evita:**

* Comprobar tipos para decidir comportamiento que podría ser un método virtual.
* El cast directo `(T)` cuando el tipo no está garantizado.

-----

## Resumen en 5 líneas

1. Polimorfismo: la misma llamada (`a.HacerSonido()`) ejecuta la implementación del tipo real del objeto.
2. El tipo estático decide qué miembros puedes usar; el tipo dinámico, qué implementación virtual se ejecuta.
3. Upcasting (derivado → base) es implícito y seguro; downcasting (base → derivado) es explícito y puede fallar.
4. `is Tipo t` comprueba y convierte en un paso; `as` devuelve `null` si falla; `(Tipo)` lanza `InvalidCastException`.
5. Muchas comprobaciones de tipo son la señal de que falta un método virtual.

-----

## Para profundizar

<details>
<summary>GetType() frente a is</summary>

`is` acepta el tipo y sus derivados; `GetType()` devuelve el tipo exacto:

```csharp
Animal a = new Perro("Firulais");
Console.WriteLine(a is Animal);                    // True
Console.WriteLine(a.GetType() == typeof(Animal));  // False
Console.WriteLine(a.GetType() == typeof(Perro));   // True
```

Usa `is` para preguntar "¿puedo tratarlo como X?" y `GetType()` solo si de verdad necesitas el tipo exacto (algo poco frecuente).

</details>

<details>
<summary>Covarianza y contravarianza (una primera mirada)</summary>

Un `IEnumerable<Perro>` se puede usar donde se espera un `IEnumerable<Animal>`, porque solo se leen elementos (covarianza, `out T`). Un `Action<Animal>` se puede usar donde se espera un `Action<Perro>`, porque una acción que sabe tratar cualquier animal sabe tratar un perro (contravarianza, `in T`). Pero `List<Perro>` **no** se puede usar como `List<Animal>`: si se pudiera, alguien podría agregarle un `Gato`. Se profundiza en [Restricciones y varianza](../05-tipos-avanzados/05-Restricciones%20y%20varianza.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

El polimorfismo permite que objetos de distintas clases se usen a través de un tipo común y que cada uno responda a su manera a la misma llamada, por ejemplo con métodos `virtual` y `override` o con interfaces. El upcasting, de una clase derivada a la base, es implícito. El downcasting, de base a derivada, es explícito y puede fallar, por eso se verifica con `is` o con `as`.

### Respuesta ampliada (semi-senior)

El polimorfismo de subtipos se basa en el despacho dinámico: el tipo estático de la referencia determina qué miembros son accesibles y el tipo en tiempo de ejecución determina qué implementación virtual se invoca. Es lo que permite cumplir el principio abierto/cerrado, reemplazando las cadenas de comprobaciones de tipo por métodos sobrescritos. El downcast con `(T)` lanza `InvalidCastException`; `as` devuelve `null` y solo aplica a tipos de referencia o anulables; el pattern matching (`is T t`, `switch` con patrones de tipo y de propiedades) combina la comprobación y la extracción. Además existen el polimorfismo ad hoc (sobrecargas, resueltas al compilar) y el paramétrico (genéricos).

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `is` y `as`?**
`is` devuelve `bool` (y con patrón, además asigna una variable). `as` devuelve la referencia convertida o `null`.

**2. ¿Qué pasa si haces un downcast incorrecto con `(T)`?**
Se lanza `InvalidCastException` al ejecutar.

**3. ¿La sobrecarga es polimorfismo?**
Sí, polimorfismo ad hoc o estático, resuelto al compilar. El polimorfismo "clásico" de la POO es el de subtipos, resuelto al ejecutar con `virtual`/`override`.

**4. ¿Cómo eliminarías un `switch` sobre el tipo de un objeto?**
Moviendo el comportamiento de cada caso a un método virtual (o a un miembro de interfaz) implementado en cada clase.

-----

## Práctica

**Ejercicio 1.** ¿Qué imprime cada línea? Si alguna no compila o falla al ejecutar, indícalo.

```csharp
Animal a = new Gato("Michi");
Console.WriteLine(a.HacerSonido());        // (1)
Console.WriteLine(a is Gato);              // (2)
Console.WriteLine(a is Perro);             // (3)
Perro? p = a as Perro;
Console.WriteLine(p is null);              // (4)
Perro p2 = (Perro)a;                       // (5)
```

(Usa las clases `Animal`, `Perro` y `Gato` de la lección).

<details>
<summary>Solución</summary>

1. `Miau`: método virtual, decide el tipo real.
2. `True`.
3. `False`.
4. `True`: `as` devolvió `null`.
5. Compila, pero lanza `InvalidCastException` al ejecutar.

</details>

**Ejercicio 2.** Refactoriza este método para eliminar las comprobaciones de tipo usando polimorfismo:

```csharp
static double CalcularEnvio(object paquete)
{
    if (paquete is Sobre) return 5;
    if (paquete is Caja c) return 10 + c.PesoKg * 2;
    if (paquete is Pallet p) return 100 + p.PesoKg * 0.5;
    throw new ArgumentException("Tipo de paquete desconocido");
}
```

<details>
<summary>Solución</summary>

```csharp
Paquete[] envios = { new Sobre(), new Caja(3), new Pallet(400) };
foreach (var e in envios)
    Console.WriteLine($"{e.GetType().Name}: {e.CalcularEnvio()}");
// Sobre: 5
// Caja: 16
// Pallet: 300

abstract class Paquete
{
    public double PesoKg { get; }
    protected Paquete(double pesoKg) => PesoKg = pesoKg;
    public abstract double CalcularEnvio();
}

class Sobre : Paquete
{
    public Sobre() : base(0.1) { }
    public override double CalcularEnvio() => 5;
}

class Caja : Paquete
{
    public Caja(double pesoKg) : base(pesoKg) { }
    public override double CalcularEnvio() => 10 + PesoKg * 2;
}

class Pallet : Paquete
{
    public Pallet(double pesoKg) : base(pesoKg) { }
    public override double CalcularEnvio() => 100 + PesoKg * 0.5;
}
```

Ahora un tipo de paquete nuevo es una clase nueva, y ya no existe el caso "tipo desconocido" en ejecución: el compilador obliga a implementar `CalcularEnvio`.

</details>

-----

## Siguiente lección

[La clase Object](10-La%20clase%20Object.md)
