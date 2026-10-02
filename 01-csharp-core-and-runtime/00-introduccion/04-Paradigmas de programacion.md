# Paradigmas de programación

## En una frase

Un paradigma es una forma de pensar y organizar el código; C# es **multiparadigma**: su base es la programación orientada a objetos, pero combina programación estructurada, funcional y orientada a eventos según lo que resuelve mejor cada problema.

-----

## Antes de empezar

Conviene que ya sepas:

* Escribir y ejecutar un programa de consola, como en [Tu primer programa](03-Tu%20primer%20programa.md).

Esta lección es un **mapa**: muestra código que todavía no conoces (clases, lambdas, eventos). No hace falta entenderlo línea por línea ahora; cada parte tiene su lección más adelante. El objetivo es reconocer los estilos.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Paradigma:** un estilo o modelo para estructurar programas.
* **Imperativo:** describes **cómo** hacer algo, paso a paso.
* **Declarativo:** describes **qué** quieres obtener y el lenguaje decide cómo.
* **Función pura:** una función que, con la misma entrada, siempre devuelve lo mismo y no modifica nada fuera de ella.
* **Efecto secundario:** cualquier cambio fuera de la función: modificar una variable externa, escribir en consola o en un archivo.
* **Inmutabilidad:** no modificar datos existentes, sino crear datos nuevos.
* **Aspecto transversal (*cross-cutting concern*):** lógica que se repite en muchas partes del sistema, como logging, seguridad o medición de tiempos.

-----

## El problema

El mismo problema se puede resolver de muchas formas, y no todas escalan igual. Por ejemplo: "sumar los cuadrados de los números pares de una lista".

* Puedes recorrer la lista con un bucle, preguntar si cada número es par y acumular en una variable.
* O puedes describir la operación: *filtrar pares → elevar al cuadrado → sumar*.

Las dos funcionan. Elegir bien depende de qué es más claro, más fácil de probar y más difícil de romper. Conocer los paradigmas te da ese vocabulario para decidir y para leer código ajeno.

-----

## Cómo funciona

### Imperativo frente a declarativo

```csharp
int[] numeros = { 1, 2, 3, 4, 5, 6 };

// Imperativo: CÓMO hacerlo, paso a paso
int suma = 0;
foreach (int n in numeros)
{
    if (n % 2 == 0)
    {
        suma += n * n;
    }
}

// Declarativo: QUÉ quiero (con LINQ)
int suma2 = numeros.Where(n => n % 2 == 0).Select(n => n * n).Sum();
```

Las dos dan `56` (4 + 16 + 36). Casi todos los paradigmas caen en uno de estos dos grandes grupos.

### 1. Programación estructurada (imperativa)

Organiza el programa con tres estructuras de control y evita los saltos arbitrarios (`goto`):

1. **Secuencia:** las instrucciones se ejecutan una tras otra.
2. **Selección:** se elige un camino (`if`, `switch`).
3. **Iteración:** se repite un bloque (`for`, `while`).

A eso se suma la **modularización**: dividir el programa en métodos.

```csharp
int numero = 10;                          // secuencia

if (numero > 5)                           // selección
    Console.WriteLine("Mayor que 5");

for (int i = 1; i <= 3; i++)              // iteración
    Console.WriteLine($"Vuelta {i}");

Console.WriteLine(Sumar(5, 7));           // modularización

static int Sumar(int a, int b) => a + b;
```

Es la base de todo lo demás: dentro de un método de una clase o de una función pura sigues escribiendo secuencias, decisiones y bucles. Se estudia en [Control de flujo](../02-control-de-flujo/README.md).

### 2. Programación orientada a objetos (POO)

Organiza el código en **objetos** que agrupan **datos** (estado) y **comportamiento** (métodos). Las **clases** son los moldes de esos objetos. Se apoya en cuatro pilares: **abstracción, encapsulación, herencia y polimorfismo**.

```csharp
abstract class Vehiculo
{
    public string Marca { get; }
    protected int Velocidad { get; set; }

    protected Vehiculo(string marca) => Marca = marca;

    public abstract void Acelerar();                       // cada vehículo acelera distinto
    public override string ToString() => $"{Marca} a {Velocidad} km/h";
}

class Auto : Vehiculo
{
    public Auto(string marca) : base(marca) { }
    public override void Acelerar() => Velocidad += 15;
}

class Moto : Vehiculo
{
    public Moto(string marca) : base(marca) { }
    public override void Acelerar() => Velocidad += 25;
}
```

```csharp
Vehiculo[] garage = { new Auto("Toyota"), new Moto("Honda") };

foreach (Vehiculo v in garage)
{
    v.Acelerar();               // cada uno ejecuta su propia versión (polimorfismo)
    Console.WriteLine(v);
}
// Toyota a 15 km/h
// Honda a 25 km/h
```

| Pilar | En el ejemplo |
| --- | --- |
| Abstracción | `Vehiculo` describe lo esencial de cualquier vehículo. |
| Encapsulación | `Velocidad` es `protected`: solo la clase y sus hijas la modifican. |
| Herencia | `Auto` y `Moto` heredan `Marca` y `ToString()` de `Vehiculo`. |
| Polimorfismo | `v.Acelerar()` ejecuta la versión de `Auto` o de `Moto` según el objeto real. |

Es el paradigma central de C#: casi toda la librería estándar y los frameworks (ASP.NET Core, Entity Framework) están diseñados con él. Se estudia en [Programación orientada a objetos](../04-poo/README.md).

### 3. Programación funcional

Construye el programa combinando **funciones puras** y **datos inmutables**. Sus ideas clave:

* **Funciones puras:** misma entrada, misma salida, sin efectos secundarios.
* **Inmutabilidad:** en lugar de modificar, se crean valores nuevos.
* **Funciones de orden superior:** funciones que reciben o devuelven otras funciones.
* **Composición:** encadenar transformaciones pequeñas.

```csharp
var numeros = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

int resultado = numeros
    .Where(n => n % 2 == 0)              // filtrar (recibe una función)
    .Select(n => n * n)                  // transformar
    .Aggregate(0, (acc, n) => acc + n);  // reducir

Console.WriteLine(resultado);            // 220
Console.WriteLine(Factorial(5));         // 120

// Función pura: no depende ni modifica nada externo
static int Factorial(int n) => n <= 1 ? 1 : n * Factorial(n - 1);
```

`Where`, `Select` y `Aggregate` no modifican la lista original: producen resultados nuevos. C# incorpora este estilo mediante **LINQ**, **lambdas**, **records** y **pattern matching**. Se ve en [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md) y en [LINQ](../07-linq/README.md).

### 4. Programación reactiva y orientada a eventos

En lugar de preguntar todo el tiempo "¿cambió algo?", el código **se suscribe** y **reacciona** cuando ocurre algo (un clic, un dato nuevo, un mensaje).

```csharp
var sensor = new SensorTemperatura();

// Suscripción: "cuando cambie la temperatura, ejecuta esto"
sensor.TemperaturaCambiada += (_, temp) =>
{
    Console.WriteLine($"Temperatura: {temp} °C");
    if (temp > 30) Console.WriteLine("¡Alerta! Temperatura alta");
};

sensor.Medir(25.5);
sensor.Medir(32.1);

class SensorTemperatura
{
    public event EventHandler<double>? TemperaturaCambiada;

    public void Medir(double valor) => TemperaturaCambiada?.Invoke(this, valor);
}
```

C# trae **eventos** y **delegados** de forma nativa. La programación reactiva completa, con flujos de datos que se filtran y transforman en el tiempo, se hace con la librería **Reactive Extensions** (`System.Reactive`). Se usa en interfaces gráficas, IoT y procesamiento en tiempo real. Los delegados y los eventos se estudian en [Delegados y eventos](../09-delegados-y-eventos/README.md).

### 5. Programación orientada a aspectos (AOP)

Separa los **aspectos transversales** (logging, seguridad, transacciones, medición de tiempos) de la lógica de negocio, para no repetirlos en cada método.

Sin AOP, cada método mezcla dos responsabilidades:

```csharp
public decimal ProcesarPago(decimal monto)
{
    Console.WriteLine("[LOG] Inicio ProcesarPago");   // transversal
    var reloj = Stopwatch.StartNew();                  // transversal

    decimal total = monto * 1.18m;                     // negocio

    Console.WriteLine($"[LOG] Fin en {reloj.ElapsedMilliseconds} ms"); // transversal
    return total;
}
```

Con AOP, la lógica de negocio queda limpia y el aspecto se aplica desde afuera:

```csharp
[Loguear, MedirTiempo]               // el "cómo" del logging vive en otro lugar
public decimal ProcesarPago(decimal monto) => monto * 1.18m;
```

Vocabulario de AOP:

| Término | Significado |
| --- | --- |
| Aspecto | El módulo que encapsula el comportamiento transversal. |
| Advice (consejo) | El código que se ejecuta (antes, después o ante una excepción). |
| Pointcut (punto de corte) | Dónde se aplica el aspecto. |
| Weaving (entrelazado) | El proceso de insertar el aspecto en el código. |

C# no trae AOP en el lenguaje. Se logra con librerías (Metalama, Castle DynamicProxy) o, mucho más habitual en el día a día, con mecanismos de los frameworks que cumplen el mismo objetivo: **middleware** y **filtros** en ASP.NET Core, **decoradores** con inyección de dependencias, o los *behaviors* de pipeline de MediatR.

-----

## Ejemplo completo

El mismo problema, "obtener los nombres de los productos con stock, en mayúsculas", resuelto en dos estilos dentro de un mismo programa:

```csharp
var productos = new List<Producto>
{
    new("teclado", 10),
    new("mouse", 0),
    new("monitor", 3),
};

// Estilo estructurado (imperativo)
var conStock1 = new List<string>();
foreach (var p in productos)
{
    if (p.Stock > 0)
    {
        conStock1.Add(p.Nombre.ToUpper());
    }
}

// Estilo funcional (declarativo)
var conStock2 = productos
    .Where(p => p.Stock > 0)
    .Select(p => p.Nombre.ToUpper())
    .ToList();

Console.WriteLine(string.Join(", ", conStock1)); // TECLADO, MONITOR
Console.WriteLine(string.Join(", ", conStock2)); // TECLADO, MONITOR

// POO: el dato modelado como tipo (un record, se verá más adelante)
record Producto(string Nombre, int Stock);
```

Tres paradigmas conviven en 25 líneas: un tipo de POO (`Producto`), un bucle estructurado y una consulta funcional. Así se escribe C# real.

-----

## Errores comunes

**1. Creer que "orientado a objetos" significa "todo debe ser una clase con herencia".**
Qué pasa: jerarquías profundas y rígidas, difíciles de cambiar.
Por qué: la POO también es encapsulación y composición, no solo herencia.
Arreglo: prefiere composición e interfaces; usa herencia solo cuando hay una relación "es un" real. Se discute en [Herencia](../04-poo/06-Herencia.md).

**2. Pensar que usar LINQ hace al código "funcional".**
Qué pasa: lambdas que modifican variables externas o escriben en base de datos dentro de un `Select`.
Por qué: lo funcional no es la sintaxis, es evitar efectos secundarios.
Arreglo: mantén las lambdas de LINQ sin efectos secundarios; los efectos van fuera, al final.

**3. Elegir un paradigma "por moda".**
Qué pasa: código artificialmente complicado (una cadena de LINQ ilegible donde un `foreach` era más claro).
Por qué: ningún paradigma es mejor en todos los casos.
Arreglo: elige lo que comunique mejor la intención a quien lea el código.

-----

## Según la versión de C#

C# nació en 2002 como lenguaje orientado a objetos y fue sumando herramientas funcionales:

* **C# 3:** lambdas y LINQ.
* **C# 7–9:** tuplas, pattern matching, funciones locales, records e `init`.
* **C# 8:** expresiones `switch`.
* **C# 12:** expresiones de colección (`[1, 2, 3]`) y constructores primarios en clases.

El C# moderno se escribe mezclando estilos con mucha más naturalidad que el C# de 2005.

-----

## Cuándo sí y cuándo no

| Paradigma | Úsalo para | Ten en cuenta |
| --- | --- | --- |
| Estructurado | La lógica dentro de cualquier método. | Por sí solo no organiza sistemas grandes. |
| POO | Modelar entidades con estado y reglas (dominio, servicios). | La herencia profunda vuelve rígido el código. |
| Funcional | Transformar datos, reglas sin estado, consultas. | Sin cuidado, muchas cadenas pueden ser difíciles de depurar. |
| Eventos / reactivo | Interfaces, notificaciones, flujos en tiempo real. | El flujo es menos lineal: cuesta seguirlo al depurar. |
| AOP | Logging, auditoría, seguridad, transacciones. | El comportamiento "oculto" puede sorprender a quien no lo conoce. |

-----

## Resumen en 5 líneas

1. Un paradigma es una forma de organizar el código; C# es multiparadigma.
2. Estructurado: secuencia, selección e iteración, la base de cualquier método.
3. POO: objetos con estado y comportamiento, con 4 pilares (abstracción, encapsulación, herencia, polimorfismo).
4. Funcional: funciones puras, inmutabilidad y composición (LINQ, lambdas, records).
5. Eventos/reactivo reaccionan a cambios; AOP separa los aspectos transversales.

-----

## Para profundizar

<details>
<summary>Ejemplo con Reactive Extensions (Rx)</summary>

Con el paquete `System.Reactive` (`dotnet add package System.Reactive`), los eventos se tratan como flujos que se pueden filtrar y transformar, igual que una colección con LINQ:

```csharp
using System.Reactive.Linq;
using System.Reactive.Subjects;

var temperaturas = new Subject<double>();

using var suscripcion = temperaturas
    .Where(t => t > 30)                      // solo temperaturas altas
    .Select(t => $"ALERTA: {t} °C")          // transformar a mensaje
    .DistinctUntilChanged()                  // ignorar repetidos consecutivos
    .Subscribe(Console.WriteLine);

temperaturas.OnNext(25.5);
temperaturas.OnNext(32.1);   // ALERTA: 32.1 °C
temperaturas.OnNext(35.0);   // ALERTA: 35 °C
```

Operadores como `Throttle` o `Buffer` agregan control del tiempo (limitar la frecuencia, agrupar por ventanas).

</details>

<details>
<summary>AOP con un decorador (sin librerías externas)</summary>

La forma más común de aplicar un aspecto en C# moderno es el **patrón decorador** con una interfaz:

```csharp
interface IPagos
{
    decimal Procesar(decimal monto);
}

class Pagos : IPagos
{
    public decimal Procesar(decimal monto) => monto * 1.18m;   // solo negocio
}

class PagosConLog : IPagos
{
    private readonly IPagos _interno;
    public PagosConLog(IPagos interno) => _interno = interno;

    public decimal Procesar(decimal monto)
    {
        Console.WriteLine($"[LOG] Procesar({monto})");
        var resultado = _interno.Procesar(monto);
        Console.WriteLine($"[LOG] Resultado: {resultado}");
        return resultado;
    }
}

IPagos pagos = new PagosConLog(new Pagos());
pagos.Procesar(100m);
```

El código de negocio (`Pagos`) no sabe que existe el logging. Con inyección de dependencias, el decorador se registra una vez y se aplica en toda la app.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un paradigma es un estilo de programación. C# es multiparadigma: principalmente orientado a objetos, con clases, herencia y polimorfismo, pero también soporta programación funcional con LINQ y lambdas, y programación orientada a eventos con delegados y eventos.

### Respuesta ampliada (semi-senior)

La programación imperativa describe cómo hacer algo y la declarativa describe qué se quiere. En C#, la POO estructura el dominio y los servicios; el estilo funcional, con LINQ, lambdas, records, pattern matching e inmutabilidad, reduce los efectos secundarios y facilita las pruebas; los eventos y Rx modelan flujos asíncronos. Los aspectos transversales no se resuelven en el lenguaje, sino con middleware, filtros, decoradores o behaviors de pipeline. Lo importante es elegir el estilo que comunica mejor la intención y no forzar uno solo.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es una función pura y por qué importa?**
Una función sin efectos secundarios y determinista. Es fácil de probar (no necesita preparar estado), de paralelizar y de razonar.

**2. ¿Cuáles son los cuatro pilares de la POO?**
Abstracción, encapsulación, herencia y polimorfismo.

**3. ¿C# es un lenguaje funcional?**
No en sentido estricto (como F# o Haskell), pero tiene muchas características funcionales: lambdas, funciones de orden superior, LINQ, records inmutables, pattern matching y expresiones `switch`.

**4. ¿Cómo implementarías logging transversal en una API .NET?**
Con un middleware o un filtro en ASP.NET Core, o con un decorador registrado en el contenedor de dependencias, para no repetir el código en cada endpoint.

-----

## Práctica

**Ejercicio 1.** Clasifica cada fragmento como **imperativo** o **declarativo**:

```csharp
// A
var mayores = edades.Where(e => e >= 18).ToList();

// B
var mayores2 = new List<int>();
for (int i = 0; i < edades.Length; i++)
    if (edades[i] >= 18) mayores2.Add(edades[i]);
```

<details>
<summary>Solución</summary>

A es **declarativo**: describe qué quieres (los mayores de edad). B es **imperativo**: describe cómo recorrer, comparar y agregar.

</details>

**Ejercicio 2.** ¿Esta función es pura? Justifica.

```csharp
int contador = 0;
int Incrementar(int x)
{
    contador++;
    return x + 1;
}
```

<details>
<summary>Solución</summary>

**No es pura**: aunque siempre devuelve `x + 1`, modifica `contador`, una variable externa. Eso es un efecto secundario.

</details>

-----

## Siguiente lección

Terminaste la introducción. Continúa con [Tipos y variables](../01-tipos-y-variables/README.md).
