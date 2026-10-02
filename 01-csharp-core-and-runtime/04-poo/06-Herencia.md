# Herencia

## En una frase

La herencia permite que una clase (**derivada** o subclase) reciba los miembros de otra (**base** o superclase) con la sintaxis `class Sedan : Vehiculo`, para reutilizar el código común y modelar relaciones del tipo "**es un**".

-----

## Antes de empezar

Conviene que ya sepas:

* Crear clases con propiedades y constructores, de [Constructores y this](04-Constructores%20y%20this.md).
* Qué hace `protected`, de [Modificadores de acceso y encapsulación](02-Modificadores%20de%20acceso%20y%20encapsulacion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Herencia:** mecanismo por el que una clase obtiene los miembros de otra.
* **Clase base (superclase, padre):** la clase de la que se hereda.
* **Clase derivada (subclase, hija):** la clase que hereda.
* **Jerarquía de herencia:** el árbol formado por clases base y derivadas.
* **`base`:** palabra clave para acceder a miembros o constructores de la clase base.
* **`protected`:** accesible en la clase y en sus derivadas.
* **`sealed`:** impide que una clase sea heredada.
* **Composición:** construir una clase usando objetos de otras clases como campos ("tiene un"), en lugar de heredar.

-----

## El problema

Tienes dos clases, `Sedan` y `Camion`:

```csharp
class Sedan
{
    public string Patente { get; }
    public double Velocidad { get; private set; }
    public int Ruedas => 4;

    public Sedan(string patente) => Patente = patente;
    public void Acelerar() => Velocidad += 5;
    public void Frenar() => Velocidad = Math.Max(0, Velocidad - 5);
    public void TocarBocina() => Console.WriteLine("¡Bip!");
}

class Camion
{
    public string Patente { get; }
    public double Velocidad { get; private set; }
    public int Ruedas => 8;

    public Camion(string patente) => Patente = patente;
    public void Acelerar() => Velocidad += 5;      // copiado
    public void Frenar() => Velocidad = Math.Max(0, Velocidad - 5);   // copiado
    public void TocarBocina() => Console.WriteLine("¡Bip!");           // copiado
}
```

El código duplicado trae dos problemas:

1. **Pérdida de tiempo:** cada cambio se hace en varios lugares.
2. **Errores:** tarde o temprano corriges `Acelerar` en `Sedan` y te olvidas de `Camion`. Con dos clases ya pasa; con veinte, es seguro.

Además, no hay forma de decir "dame cualquier vehículo": `Sedan` y `Camion` no tienen relación para el compilador.

-----

## Cómo funciona

### Declarar una clase base y una derivada

Se extrae lo común a una clase base y las clases específicas heredan de ella con `:`:

```csharp
class Vehiculo
{
    public string Patente { get; }
    public double Velocidad { get; private set; }

    public Vehiculo(string patente) => Patente = patente;

    public void Acelerar() => Velocidad += 5;
    public void Frenar() => Velocidad = Math.Max(0, Velocidad - 5);
    public void TocarBocina() => Console.WriteLine("¡Bip!");
}

class Sedan : Vehiculo
{
    public int Ruedas => 4;
    public Sedan(string patente) : base(patente) { }
}

class Camion : Vehiculo
{
    public int Ruedas => 8;
    public double CargaMaximaKg { get; }

    public Camion(string patente, double carga) : base(patente)
    {
        CargaMaximaKg = carga;
    }
}
```

```csharp
var s = new Sedan("ABC-123");
s.Acelerar();                       // heredado de Vehiculo
s.TocarBocina();                    // heredado
Console.WriteLine(s.Velocidad);     // 5
Console.WriteLine(s.Ruedas);        // 4 (propio de Sedan)

var c = new Camion("XYZ-999", 12_000);
Console.WriteLine(c.CargaMaximaKg); // propio de Camion
```

`Sedan` "es un" `Vehiculo`: tiene todo lo que tiene un vehículo, más lo suyo. La lógica de `Acelerar` vive en **un solo lugar**.

```text
              Vehiculo
          (Patente, Velocidad,
           Acelerar, Frenar)
             ▲          ▲
             │          │
          Sedan       Camion
        (Ruedas=4)  (Ruedas=8, CargaMaximaKg)
```

### Qué se hereda y qué no

* Se heredan los campos, propiedades, métodos y eventos.
* Los miembros `private` de la base **existen** en el objeto derivado, pero la clase derivada **no puede usarlos** directamente.
* Los **constructores no se heredan**: cada clase declara los suyos y llama a uno de la base.

### Herencia simple: una sola clase base

Una clase en C# puede heredar de **una sola** clase:

```csharp
class Anfibio : Vehiculo, Barco { }   // error CS1721: una clase no puede tener varias clases base
```

Pero la cadena puede tener varios niveles: `CamionetaPickup : Camion : Vehiculo`. Y todas las clases, en última instancia, heredan de `object` (se ve en [La clase Object](10-La%20clase%20Object.md)).

Para que una clase "cumpla" varios contratos se usan **interfaces**, que se pueden implementar en cualquier cantidad. Si hay clase base e interfaces, la clase base va **primero**:

```csharp
class Sedan : Vehiculo, IAsegurable, IComparable<Sedan> { }   // base, luego interfaces
```

Se estudian en [Interfaces](08-Interfaces.md).

### `protected`: acceso para la familia

En la clase base, `Velocidad` tiene `private set`. ¿Qué pasa si `Camion` quiere limitar su propia velocidad?

```csharp
class Camion : Vehiculo
{
    public void Limitar() => Velocidad = 80;   // error CS0272: el accesor set es inaccesible
}
```

* `public set` sería demasiado abierto: cualquiera podría cambiar la velocidad.
* `private set` es demasiado cerrado: ni las hijas pueden.

`protected` es el punto medio: accesible en la clase y **en sus derivadas**, pero no desde afuera.

```csharp
class Vehiculo
{
    public double Velocidad { get; protected set; }
}

class Camion : Vehiculo
{
    public void Limitar() => Velocidad = Math.Min(Velocidad, 80);   // ✅
}

var c = new Camion("XYZ-999", 12_000);
c.Velocidad = 200;   // ❌ CS0272: desde afuera sigue sin poder escribirse
```

### Constructores y `base(...)`

Para construir un `Sedan`, primero hay que construir la parte `Vehiculo`. El constructor de la derivada llama al de la base con `: base(...)`:

```csharp
class Sedan : Vehiculo
{
    public Sedan(string patente) : base(patente)   // ejecuta Vehiculo(patente) PRIMERO
    {
        Console.WriteLine("Sedan creado");          // y después esto
    }
}
```

Si no escribes `: base(...)`, C# llama **implícitamente** al constructor sin parámetros de la base:

```csharp
class Sedan : Vehiculo
{
    public Sedan(string patente)        // equivale a: public Sedan(string patente) : base()
    {
    }
}
```

Si la base **no tiene** constructor sin parámetros (como `Vehiculo`, que exige `patente`), eso no compila:

```text
error CS7036: There is no argument given that corresponds to the required parameter 'patente' of 'Vehiculo.Vehiculo(string)'
```

El orden de construcción es siempre **de la base hacia la derivada**: primero `object`, luego `Vehiculo`, luego `Sedan`.

### `base.Miembro`: usar la versión de la clase base

Dentro de la derivada, `base.` se refiere a los miembros de la clase base. Es especialmente útil al **sobrescribir** un método para extender su comportamiento en lugar de reemplazarlo:

```csharp
class Vehiculo
{
    public virtual string Describir() => $"Vehículo {Patente}";
    // ...
}

class Camion : Vehiculo
{
    public override string Describir() => $"{base.Describir()} con carga máxima {CargaMaximaKg} kg";
}
```

`virtual` y `override` son el tema de la [próxima lección](07-Virtual%20override%20y%20clases%20abstractas.md).

### `sealed`: cerrar la herencia

Si una clase no está diseñada para ser extendida, márcala `sealed`:

```csharp
sealed class Configuracion { }

class MiConfig : Configuracion { }   // error CS0509: no se puede derivar del tipo sellado 'Configuracion'
```

`string` es una clase sellada: no puedes heredar de ella.

### Herencia ("es un") frente a composición ("tiene un")

La herencia es la relación **más fuerte** entre dos clases: la derivada depende de cada detalle de la base. Úsala solo cuando la frase "X **es un** Y" es verdadera siempre:

* Un `Sedan` **es un** `Vehiculo`. ✅ Herencia.
* Un `Vehiculo` **tiene un** `Motor`. ✅ Composición: `Motor` es una propiedad, no una clase base.
* Un `Pato` **es un**... ¿`Ave` o `ObjetoQueVuela`? Si heredas de `ObjetoQueVuela` y después aparece un pingüino, tienes un problema. Las capacidades se modelan mejor con interfaces.

```csharp
// Composición: Auto TIENE un Motor
class Motor
{
    public int Caballos { get; }
    public Motor(int caballos) => Caballos = caballos;
    public void Encender() => Console.WriteLine("Brrrm");
}

class Auto : Vehiculo
{
    private readonly Motor _motor;
    public Auto(string patente, Motor motor) : base(patente) => _motor = motor;
    public void Arrancar() => _motor.Encender();
}
```

Principio conocido: **"prefiere la composición sobre la herencia"**. No significa "nunca heredes", sino "hereda solo cuando la relación es realmente es-un y la base está pensada para ser extendida".

-----

## Ejemplo completo

```csharp
var empleados = new Empleado[]
{
    new Empleado("Ana", 3000m),
    new Gerente("Luis", 5000m, 1500m),
    new Practicante("Eva", 1200m, "Universidad Nacional")
};

foreach (var e in empleados)
{
    Console.WriteLine(e.Describir());
}

Console.WriteLine($"Planilla total: {empleados.Sum(e => e.SueldoTotal()):N2}");

class Empleado
{
    public string Nombre { get; }
    protected decimal SueldoBase { get; }

    public Empleado(string nombre, decimal sueldoBase)
    {
        Nombre = nombre;
        SueldoBase = sueldoBase;
    }

    public virtual decimal SueldoTotal() => SueldoBase;

    public virtual string Describir() => $"{Nombre}: {SueldoTotal():N2}";
}

class Gerente : Empleado
{
    public decimal Bono { get; }

    public Gerente(string nombre, decimal sueldoBase, decimal bono) : base(nombre, sueldoBase)
    {
        Bono = bono;
    }

    public override decimal SueldoTotal() => SueldoBase + Bono;    // usa el protected de la base

    public override string Describir() => $"{base.Describir()} (gerente, bono {Bono:N2})";
}

sealed class Practicante : Empleado
{
    public string Universidad { get; }

    public Practicante(string nombre, decimal sueldoBase, string universidad) : base(nombre, sueldoBase)
    {
        Universidad = universidad;
    }

    public override string Describir() => $"{base.Describir()} (practicante de {Universidad})";
}
```

Salida:

```text
Ana: 3,000.00
Luis: 6,500.00 (gerente, bono 1,500.00)
Eva: 1,200.00 (practicante de Universidad Nacional)
Planilla total: 10,700.00
```

Un array de `Empleado` contiene objetos de tres clases distintas, porque un `Gerente` **es un** `Empleado`. `Sum` es de LINQ. Lo de `virtual`/`override` se ve a fondo en la próxima lección; aquí solo muestra cómo `base.Describir()` reutiliza la versión de la base.

-----

## Errores comunes

**1. Heredar de dos clases.**
Qué pasa: `error CS1721: Class 'Anfibio' cannot have multiple base classes: 'Vehiculo' and 'Barco'`.
Por qué: C# solo admite herencia simple de clases.
Arreglo: hereda de una y usa interfaces o composición para lo demás.

**2. La base no tiene constructor sin parámetros y no llamas a `base(...)`.**
Qué pasa: `error CS7036: There is no argument given that corresponds to the required parameter 'patente' of 'Vehiculo.Vehiculo(string)'`.
Por qué: sin `: base(...)`, se intenta llamar a `base()`.
Arreglo: `public Sedan(string patente) : base(patente) { }`.

**3. Usar un miembro `private` de la base desde la derivada.**
Qué pasa: `error CS0122: 'Vehiculo._velocidad' is inaccessible due to its protection level`.
Por qué: `private` es solo para la clase que lo declara.
Arreglo: usa `protected` si las derivadas lo necesitan, o un método o propiedad de la base.

**4. Poner una interfaz antes de la clase base.**
Qué pasa: `error CS1722: Base class 'Vehiculo' must come before any interfaces`.
Por qué: la sintaxis exige la clase base primero.
Arreglo: `class Sedan : Vehiculo, IAsegurable`.

**5. Heredar de una clase sellada.**
Qué pasa: `error CS0509: 'MiConfig': cannot derive from sealed type 'Configuracion'`.
Por qué: la clase fue cerrada a la herencia a propósito.
Arreglo: usa composición: guarda un objeto de esa clase como campo.

**6. Declarar en la derivada un método con el mismo nombre sin `override`.**
Qué pasa: `warning CS0108: 'Camion.Describir()' hides inherited member 'Vehiculo.Describir()'. Use the new keyword if hiding was intended.`
Por qué: estás **ocultando** el método de la base en lugar de sobrescribirlo, lo que se comporta distinto según el tipo de la variable.
Arreglo: casi siempre lo que quieres es `virtual` en la base y `override` en la derivada. Se explica en la próxima lección.

-----

## Según la versión de C#

La herencia funciona igual desde C# 1. Lo que cambió es cómo se combina con otras características:

* **C# 9:** los `record` también admiten herencia (`record Gerente : Empleado`).
* **C# 12:** las clases con constructor primario pasan los parámetros a la base así: `class Sedan(string patente) : Vehiculo(patente)`.

-----

## Cuándo sí y cuándo no

**Usa herencia cuando:**

* La relación "es un" es verdadera siempre, no solo hoy.
* La clase base está **diseñada** para ser extendida (tiene miembros `virtual` o `abstract` pensados para eso).
* Las derivadas pueden usarse en cualquier lugar donde se espera la base sin romper nada (principio de sustitución de Liskov).

**Usa composición cuando:**

* La relación es "tiene un" o "usa un".
* Solo quieres reutilizar algunos métodos de otra clase.

**Usa interfaces cuando:**

* Quieres expresar capacidades ("puede volar", "se puede guardar") que comparten clases sin relación entre sí.

**Evita:**

* Jerarquías de más de 2 o 3 niveles: son difíciles de entender y de cambiar.
* Heredar "para reutilizar un método": esa es la señal de que corresponde composición.

-----

## Resumen en 5 líneas

1. `class Derivada : Base` hereda los miembros de la base; los constructores no se heredan.
2. C# admite una sola clase base, pero muchas interfaces (la clase base va primero).
3. `protected` da acceso a las derivadas sin abrirlo al resto.
4. `: base(...)` llama al constructor de la base, que se ejecuta antes; `base.Metodo()` usa su versión del método.
5. Hereda solo si la relación es "es un"; para "tiene un", usa composición. `sealed` cierra la herencia.

-----

## Para profundizar

<details>
<summary>Principio de sustitución de Liskov (la L de SOLID)</summary>

Una clase derivada debe poder usarse en cualquier lugar donde se espera la base **sin cambiar el comportamiento correcto del programa**. El ejemplo clásico de violación: `Cuadrado : Rectangulo`. Matemáticamente un cuadrado "es un" rectángulo, pero si `Rectangulo` permite cambiar `Ancho` y `Alto` por separado, un `Cuadrado` no puede cumplir ese contrato sin romper sus propias reglas. El código que hace `r.Ancho = 5; r.Alto = 10;` y espera un área de 50 falla con un `Cuadrado`.

Moraleja: "es un" en el mundo real no garantiza "es un" en el código. Lo que importa es el **comportamiento**.

</details>

<details>
<summary>Ocultación con new</summary>

```csharp
class A { public void Hola() => Console.WriteLine("A"); }
class B : A { public new void Hola() => Console.WriteLine("B"); }

A x = new B();
x.Hola();          // A  ← decide el tipo de la VARIABLE
((B)x).Hola();     // B
```

Con `new`, el método que se ejecuta depende del tipo declarado de la variable, no del objeto real. Es casi siempre una fuente de confusión; se menciona para que lo reconozcas.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La herencia permite que una clase derive de otra con `:` y reciba sus miembros, para reutilizar código y modelar relaciones "es un". En C# solo se puede heredar de una clase, pero se pueden implementar muchas interfaces. Con `base` se llama al constructor o a los métodos de la clase base, y `protected` permite que las derivadas accedan a miembros que el resto no ve.

### Respuesta ampliada (semi-senior)

C# tiene herencia simple de implementación y herencia múltiple de interfaces. Los constructores no se heredan; la construcción va de la base a la derivada, y sin `: base(...)` explícito se invoca el constructor sin parámetros de la base. La herencia acopla fuertemente a la derivada con los detalles de la base (el problema de la "clase base frágil"), por eso se recomienda preferir la composición y reservar la herencia para jerarquías diseñadas para extenderse, respetando el principio de sustitución de Liskov. `sealed` cierra la jerarquía, lo que además permite al JIT desvirtualizar llamadas. Ocultar con `new` en lugar de `override` produce dispatch estático según el tipo de la variable.

### Preguntas frecuentes de seguimiento

**1. ¿C# admite herencia múltiple?**
De clases no. Una clase puede implementar múltiples interfaces.

**2. ¿Los constructores se heredan?**
No. Cada clase define los suyos y debe encadenar a un constructor de la base.

**3. ¿Por qué "composición sobre herencia"?**
Porque la composición acopla menos: puedes cambiar el componente sin afectar a toda la jerarquía, combinar comportamientos en tiempo de ejecución y probar las piezas por separado.

-----

## Práctica

**Ejercicio 1.** Crea una clase base `Animal` con `Nombre` (solo lectura, asignado en el constructor) y un método `Comer()` que imprima `"<Nombre> está comiendo"`. Crea `Perro` (con un método `Ladrar()`) y `Gato` (con `Maullar()`) que hereden de `Animal`. Crea un perro y un gato y llama a todos sus métodos.

<details>
<summary>Solución</summary>

```csharp
var perro = new Perro("Firulais");
var gato = new Gato("Michi");

perro.Comer();
perro.Ladrar();
gato.Comer();
gato.Maullar();

class Animal
{
    public string Nombre { get; }
    public Animal(string nombre) => Nombre = nombre;
    public void Comer() => Console.WriteLine($"{Nombre} está comiendo");
}

class Perro : Animal
{
    public Perro(string nombre) : base(nombre) { }
    public void Ladrar() => Console.WriteLine($"{Nombre}: ¡Guau!");
}

class Gato : Animal
{
    public Gato(string nombre) : base(nombre) { }
    public void Maullar() => Console.WriteLine($"{Nombre}: ¡Miau!");
}
```

</details>

**Ejercicio 2.** Para cada par, decide si corresponde **herencia** o **composición**, y justifica:

1. `Computadora` y `Procesador`.
2. `CuentaAhorro` y `CuentaBancaria`.
3. `Pedido` y `ListaDeProductos`.
4. `Administrador` y `Usuario`.

<details>
<summary>Solución</summary>

1. **Composición:** una computadora *tiene un* procesador.
2. **Herencia:** una cuenta de ahorro *es una* cuenta bancaria (si comparte el comportamiento base sin romperlo).
3. **Composición:** un pedido *tiene una* lista de productos; heredar de una lista expondría métodos como `Clear()` que un pedido no debería ofrecer.
4. **Depende:** si un administrador es un usuario con permisos extra, podría ser herencia; pero si los roles cambian en el tiempo (un usuario pasa a ser administrador), es mejor modelar el rol como un dato o una composición (`Usuario` *tiene un* `Rol`). Un objeto no puede cambiar de clase.

</details>

-----

## Siguiente lección

[Virtual, override y clases abstractas](07-Virtual%20override%20y%20clases%20abstractas.md)
