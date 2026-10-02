# Clases y objetos

## En una frase

Una **clase** es un tipo de dato que defines tú: un molde que agrupa **datos** (campos) y **comportamiento** (métodos); un **objeto** es una instancia concreta de ese molde, creada con `new`, con sus propios valores.

-----

## Antes de empezar

Conviene que ya sepas:

* Definir y llamar métodos, de [Métodos](../03-metodos/README.md).
* Que las clases son tipos de referencia, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Clase:** definición de un tipo propio con datos y comportamiento.
* **Objeto (instancia):** un ejemplar concreto creado a partir de una clase.
* **Instanciar:** crear un objeto con `new`.
* **Miembro:** cualquier elemento declarado dentro de una clase: campos, propiedades, métodos, constructores.
* **Campo:** variable que pertenece a la clase; cada objeto tiene su propia copia.
* **Método de instancia:** método que trabaja con los datos de un objeto concreto.
* **Estado:** los valores que tienen los campos de un objeto en un momento dado.
* **Notación de punto:** `objeto.miembro`, la forma de acceder a los miembros.
* **POO:** programación orientada a objetos.

-----

## El problema

Estás escribiendo un programa para gestionar reservas naturales. Cada bosque tiene un nombre, una cantidad de árboles y un área; además, puede crecer y puede sufrir un incendio. Con lo que sabes hasta ahora, tendrías algo así:

```csharp
string nombreBosque1 = "Amazonas";
int arbolesBosque1 = 1_000_000;
string nombreBosque2 = "Congo";
int arbolesBosque2 = 800_000;

static int Crecer(int arboles) => arboles + 30;
```

Con dos bosques ya es incómodo; con cien es imposible. Nada impide pasar los árboles del bosque 1 junto con el nombre del bosque 2, y la relación entre "nombre", "árboles" y "crecer" solo existe en tu cabeza.

La programación orientada a objetos propone **crear tu propio tipo**, `Bosque`, que agrupa esos datos y esas operaciones, y usarlo como cualquier otro tipo (`int`, `string`).

-----

## Cómo funciona

### Tipos que ya usaste como objetos

Ya trabajaste con objetos sin saberlo:

```csharp
string frase = "¡zoinks!";
Console.WriteLine(frase.Length);        // 8
Console.WriteLine(frase.IndexOf("k"));  // 5
```

`frase` es una **instancia** del tipo `string`. Cada string tiene su propio valor, pero todos comparten la misma "forma": una propiedad `Length` y un método `IndexOf`. Una clase te permite definir esa forma para tus propios tipos.

### Definir una clase

```csharp
class Bosque
{
}
```

* La palabra clave `class` seguida del nombre en **PascalCase** y en **singular** (`Bosque`, no `Bosques`).
* El cuerpo entre llaves contendrá los miembros.

**Dónde escribirla:** en un proyecto real, cada clase va en su propio archivo con el mismo nombre (`Bosque.cs`). En un `Program.cs` con top-level statements, las clases deben ir **después** de todas las sentencias.

```text
MiProyecto/
├── Program.cs     ← sentencias de alto nivel (usa Bosque)
└── Bosque.cs      ← class Bosque { ... }
```

### Crear objetos: instanciar con `new`

```csharp
Bosque b1 = new Bosque();
Bosque b2 = new Bosque();
var b3 = new Bosque();          // con var
Bosque b4 = new();              // "new con tipo de destino" (C# 9): el tipo ya está a la izquierda
```

`b1`, `b2`, `b3` y `b4` son **cuatro objetos distintos** del tipo `Bosque`. Se dice que "`b1` es una instancia de `Bosque`" o que "`b1` es de tipo `Bosque`".

```text
   clase Bosque  (el molde: describe la forma)
        │
        ├── new ──►  objeto b1  (un bosque concreto)
        ├── new ──►  objeto b2  (otro bosque, con sus propios datos)
        └── new ──►  objeto b3
```

La analogía clásica: la clase es el **plano** de una casa; los objetos son las **casas** construidas con ese plano. Todas tienen la misma distribución, pero cada una tiene su propio color y sus propios muebles.

### Campos: los datos de cada objeto

```csharp
class Bosque
{
    public string Nombre = "";
    public int Arboles;
}
```

Un campo se declara como una variable, pero dentro de la clase. Cada objeto tiene **su propia copia**:

```csharp
var amazonas = new Bosque();
amazonas.Nombre = "Amazonas";
amazonas.Arboles = 1_000_000;

var congo = new Bosque();
congo.Nombre = "Congo";

Console.WriteLine(amazonas.Nombre);   // Amazonas
Console.WriteLine(congo.Nombre);      // Congo
Console.WriteLine(congo.Arboles);     // 0 (valor por defecto)
```

* Se accede con **notación de punto**: `objeto.campo`.
* Si no asignas un valor, el campo tiene el **valor por defecto** de su tipo: `0`, `false`, `null`. (A diferencia de las variables locales, los campos sí se inicializan solos).
* `public` permite usar el campo desde fuera de la clase. Se explica en la [próxima lección](02-Modificadores%20de%20acceso%20y%20encapsulacion.md); por ahora, ponlo en todos los miembros.

### Métodos: el comportamiento de cada objeto

Los métodos dentro de una clase pueden leer y modificar los campos **del objeto sobre el que se llaman**:

```csharp
class Bosque
{
    public string Nombre = "";
    public int Arboles;

    public void Crecer()
    {
        Arboles += 30;
    }

    public void Quemar()
    {
        Arboles -= 20;
    }

    public string Describir() => $"{Nombre} tiene {Arboles} árboles.";
}
```

```csharp
var amazonas = new Bosque { Nombre = "Amazonas", Arboles = 100 };
var congo = new Bosque { Nombre = "Congo", Arboles = 50 };

amazonas.Crecer();          // solo crece el Amazonas
amazonas.Crecer();
congo.Quemar();

Console.WriteLine(amazonas.Describir());   // Amazonas tiene 160 árboles.
Console.WriteLine(congo.Describir());      // Congo tiene 30 árboles.
```

`Arboles` dentro de `Crecer()` se refiere a los árboles **del objeto que llamó al método**: en `amazonas.Crecer()`, son los del Amazonas. Estos métodos no llevan `static`, porque trabajan con un objeto concreto: son **métodos de instancia**.

### Inicializador de objetos

La sintaxis `new Bosque { Nombre = "Amazonas", Arboles = 100 }` crea el objeto y asigna miembros públicos en una sola expresión. Es equivalente a:

```csharp
var amazonas = new Bosque();
amazonas.Nombre = "Amazonas";
amazonas.Arboles = 100;
```

### Los objetos son referencias

Como las clases son tipos de referencia, asignar un objeto a otra variable **no lo copia**:

```csharp
var original = new Bosque { Nombre = "Amazonas" };
var mismo = original;
mismo.Nombre = "Otro";

Console.WriteLine(original.Nombre);   // Otro: es el mismo objeto
```

Y `null` significa "esta variable no apunta a ningún objeto":

```csharp
Bosque? ninguno = null;
ninguno.Crecer();     // NullReferenceException
```

### Encapsulación: la primera idea de la POO

Agrupar datos y comportamiento relacionados en un tipo es la base de la **encapsulación**, uno de los cuatro pilares de la POO (abstracción, encapsulación, herencia y polimorfismo). La encapsulación completa también implica **proteger** esos datos para que nadie los deje en un estado inválido (por ejemplo, `Arboles = -500`). Eso se ve en las próximas dos lecciones.

-----

## Ejemplo completo

```csharp
var tienda = new Inventario();

var teclado = new Producto { Nombre = "Teclado", Precio = 150m, Stock = 10 };
var mouse = new Producto { Nombre = "Mouse", Precio = 60m, Stock = 0 };

tienda.Agregar(teclado);
tienda.Agregar(mouse);

teclado.Vender(3);
mouse.Vender(1);

tienda.MostrarResumen();

class Producto
{
    public Guid Id = Guid.NewGuid();      // identificador único generado al crear el objeto
    public string Nombre = "";
    public decimal Precio;
    public int Stock;

    public bool Vender(int cantidad)
    {
        if (cantidad > Stock)
        {
            Console.WriteLine($"Sin stock suficiente de {Nombre}");
            return false;
        }

        Stock -= cantidad;
        return true;
    }

    public decimal ValorEnStock() => Precio * Stock;
}

class Inventario
{
    public List<Producto> Productos = new();

    public void Agregar(Producto p) => Productos.Add(p);

    public void MostrarResumen()
    {
        decimal total = 0;
        foreach (Producto p in Productos)
        {
            Console.WriteLine($"{p.Nombre,-10} stock: {p.Stock,3}  valor: {p.ValorEnStock(),8:N2}  id: {p.Id.ToString()[..8]}");
            total += p.ValorEnStock();
        }
        Console.WriteLine($"Valor total del inventario: {total:N2}");
    }
}
```

Salida (los `id` cambian en cada ejecución):

```text
Sin stock suficiente de Mouse
Teclado    stock:   7  valor: 1,050.00  id: 3f2a9c1e
Mouse      stock:   0  valor:     0.00  id: b71d04aa
Valor total del inventario: 1,050.00
```

Dos clases colaboran: `Inventario` contiene una lista de objetos `Producto`. `Guid.NewGuid()` genera un identificador único de 128 bits, muy usado como clave en bases de datos.

-----

## Errores comunes

**1. Usar un objeto sin crearlo.**
Qué pasa: `Bosque b; b.Crecer();` da `error CS0165: Use of unassigned local variable 'b'`; y si la variable vale `null`, `NullReferenceException` al ejecutar.
Por qué: declarar la variable no crea el objeto.
Arreglo: `Bosque b = new Bosque();`.

**2. Acceder a un miembro que no es público.**
Qué pasa: `error CS0122: 'Bosque.Arboles' is inaccessible due to its protection level`.
Por qué: los miembros son `private` por defecto.
Arreglo: márcalo `public` (o, mejor, expón una propiedad; ver las próximas lecciones).

**3. Llamar a un método de instancia sobre la clase.**
Qué pasa: `Bosque.Crecer();` da `error CS0120: An object reference is required for the non-static field, method, or property 'Bosque.Crecer()'`.
Por qué: `Crecer` necesita saber **qué** bosque crece.
Arreglo: llámalo sobre un objeto: `amazonas.Crecer();`.

**4. Declarar clases antes de las top-level statements.**
Qué pasa: `error CS8803: Top-level statements must precede namespace and type declarations`.
Por qué: en `Program.cs`, las sentencias sueltas van primero.
Arreglo: mueve las clases al final del archivo o, mejor, a su propio archivo.

**5. Creer que asignar un objeto lo copia.**
Qué pasa: cambias `copia.Nombre` y también cambia `original.Nombre`.
Por qué: las clases son tipos de referencia.
Arreglo: crea un objeto nuevo con los mismos valores si necesitas una copia independiente.

-----

## Según la versión de C#

* **C# 3:** inicializadores de objetos (`new Bosque { Nombre = "x" }`) y de colecciones.
* **C# 9:** `new()` con tipo de destino (`Bosque b = new();`) y `record` (clases pensadas para datos).
* **C# 10:** espacios de nombres por archivo (`namespace MiApp;` sin llaves).
* **C# 12:** constructores primarios en clases (se ven en [Constructores y this](04-Constructores%20y%20this.md)).

-----

## Cuándo sí y cuándo no

**Crea una clase cuando:**

* Un conjunto de datos va siempre junto y tiene operaciones propias (un pedido, un cliente, una cuenta).
* Te encuentras pasando las mismas 3 o 4 variables juntas a varios métodos.

**Considera otra opción cuando:**

* Solo agrupas datos sin comportamiento ni reglas: un `record` es más conciso.
* Es un valor pequeño e inmutable (una coordenada, un rango): puede ser un `struct`.
* Es una agrupación temporal para devolver dos valores: una tupla alcanza.

-----

## Resumen en 5 líneas

1. Una clase define un tipo propio con campos (datos) y métodos (comportamiento).
2. `new Clase()` crea un objeto; cada objeto tiene su propia copia de los campos.
3. Se accede a los miembros con notación de punto: `objeto.Campo`, `objeto.Metodo()`.
4. Los métodos de instancia trabajan sobre el objeto que los llama; no llevan `static`.
5. Las clases son tipos de referencia: asignar un objeto comparte el mismo objeto.

-----

## Para profundizar

<details>
<summary>class, struct y record: tabla de diferencias</summary>

| Tipo | Clase de tipo | `==` compara | Pensado para |
| --- | --- | --- | --- |
| `class` | Referencia | La referencia (si es el mismo objeto) | Entidades con identidad y comportamiento |
| `struct` | Valor | No definido por defecto (hay que implementarlo) | Valores pequeños: coordenadas, colores |
| `record` (o `record class`) | Referencia | Los valores de sus propiedades | Datos inmutables: DTOs, mensajes |
| `record struct` | Valor | Los valores de sus propiedades | Datos pequeños inmutables |

`struct` y `record` se estudian en [Structs y records](../05-tipos-avanzados/02-Structs%20y%20records.md).

</details>

<details>
<summary>Espacios de nombres (namespace)</summary>

En proyectos reales, las clases se agrupan en espacios de nombres para evitar choques de nombres y organizar el código:

```csharp
// Archivo Bosque.cs
namespace Reservas.Dominio;

public class Bosque
{
    public string Nombre = "";
}
```

```csharp
// Program.cs
using Reservas.Dominio;

var b = new Bosque();
```

La convención es que el espacio de nombres refleje la estructura de carpetas del proyecto.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una clase es una plantilla que define datos (campos y propiedades) y comportamiento (métodos). Un objeto es una instancia de esa clase, creada con `new`, que tiene sus propios valores. Las clases son tipos de referencia: la variable guarda una referencia al objeto en el heap.

### Respuesta ampliada (semi-senior)

Una clase define un tipo de referencia con estado (campos), comportamiento (métodos) y su API (propiedades, constructores, eventos). Cada instancia se reserva en el heap con sus propios campos de instancia, inicializados a `default` antes de ejecutar el constructor; la variable contiene una referencia, por lo que la asignación comparte la instancia y `==` compara identidad salvo que se sobrescriba. Los métodos de instancia reciben implícitamente `this`. Agrupar estado y comportamiento con sus invariantes es la encapsulación. Para tipos centrados en datos se prefieren los `record`, y los `struct` se reservan para valores pequeños e inmutables.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre clase y objeto?**
La clase es la definición (el molde); el objeto es una instancia concreta con su propio estado.

**2. ¿Dónde se guarda un objeto?**
En el heap. La variable (si es local) guarda en el stack una referencia a él.

**3. ¿Cuándo usarías un `struct` en lugar de una clase?**
Para valores pequeños, inmutables y que se comparan por contenido, donde evitar la asignación en el heap aporta rendimiento.

-----

## Práctica

**Ejercicio 1.** Crea una clase `CuentaBancaria` con los campos `Titular` (string) y `Saldo` (decimal), y los métodos `Depositar(decimal monto)`, `Retirar(decimal monto)` (que no debe permitir dejar el saldo negativo y devuelve `bool`) y `Resumen()` (que devuelve un string). Crea dos cuentas y opera con ellas.

<details>
<summary>Solución</summary>

```csharp
var ana = new CuentaBancaria { Titular = "Ana" };
var luis = new CuentaBancaria { Titular = "Luis" };

ana.Depositar(500);
luis.Depositar(100);
bool ok = luis.Retirar(300);

Console.WriteLine(ana.Resumen());                 // Ana: 500.00
Console.WriteLine(luis.Resumen());                // Luis: 100.00
Console.WriteLine($"¿Retiro de Luis OK? {ok}");   // False

class CuentaBancaria
{
    public string Titular = "";
    public decimal Saldo;

    public void Depositar(decimal monto) => Saldo += monto;

    public bool Retirar(decimal monto)
    {
        if (monto > Saldo) return false;
        Saldo -= monto;
        return true;
    }

    public string Resumen() => $"{Titular}: {Saldo:N2}";
}
```

Fíjate en el problema que todavía tiene: nada impide escribir `ana.Saldo = -1000;` desde afuera. Eso se resuelve en las dos próximas lecciones.

</details>

**Ejercicio 2.** ¿Qué imprime este código?

```csharp
var a = new Contador();
var b = a;
var c = new Contador();

a.Incrementar();
b.Incrementar();
c.Incrementar();

Console.WriteLine($"{a.Valor} {b.Valor} {c.Valor}");

class Contador
{
    public int Valor;
    public void Incrementar() => Valor++;
}
```

<details>
<summary>Solución</summary>

`2 2 1`. `a` y `b` apuntan al mismo objeto, que se incrementó dos veces. `c` es otro objeto.

</details>

-----

## Siguiente lección

[Modificadores de acceso y encapsulación](02-Modificadores%20de%20acceso%20y%20encapsulacion.md)
