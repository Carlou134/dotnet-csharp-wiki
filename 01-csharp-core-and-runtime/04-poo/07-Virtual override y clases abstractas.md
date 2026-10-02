# Virtual, override y clases abstractas

## En una frase

Un método `virtual` tiene una implementación por defecto que las derivadas **pueden** reemplazar con `override`; un método `abstract` no tiene implementación y las derivadas **deben** implementarlo; y una clase `abstract` es una base incompleta que no se puede instanciar.

-----

## Antes de empezar

Conviene que ya sepas:

* Crear clases base y derivadas, llamar a `base(...)` y usar `protected`, de [Herencia](06-Herencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`virtual`:** marca un miembro de la base que las derivadas pueden sobrescribir.
* **`override`:** marca un miembro de la derivada que reemplaza a uno `virtual` o `abstract` de la base.
* **Sobrescritura (*overriding*):** dar una nueva implementación a un miembro heredado.
* **`abstract` (miembro):** miembro sin implementación que las derivadas no abstractas deben implementar.
* **Clase abstracta:** clase marcada `abstract` que no se puede instanciar y sirve solo como base.
* **Clase concreta:** clase que sí se puede instanciar.
* **Despacho dinámico:** decidir **al ejecutar** qué implementación de un método se llama, según el tipo real del objeto.

-----

## El problema

Tienes `Vehiculo` con un método `Acelerar()` que suma 5 km/h. Llega un nuevo requisito: las motos aceleran de a 15 y los camiones de a 2. Además, cada vehículo debe poder describirse, pero **no existe** una descripción genérica razonable de "un vehículo": cada tipo se describe distinto.

Con lo que viste hasta ahora:

* Si declaras `Acelerar()` de nuevo en `Moto`, solo ocultas el de la base: cuando el código trabaja con una variable `Vehiculo`, se sigue ejecutando la versión de 5 km/h.
* Si pones un `Describir()` en `Vehiculo` que devuelve `""`, nadie te obliga a implementarlo en las derivadas, y un día alguien olvida hacerlo.

Necesitas dos cosas: que una derivada pueda **reemplazar** un comportamiento de la base (de forma que se respete aunque la variable sea de tipo base), y que la base pueda **exigir** a las derivadas que implementen algo.

-----

## Cómo funciona

### `virtual` y `override`

En la base, marca el método como `virtual`: "**este comportamiento se puede reemplazar**".

```csharp
class Vehiculo
{
    public double Velocidad { get; protected set; }

    public virtual void Acelerar()
    {
        Velocidad += 5;
    }
}
```

En la derivada, márcalo `override`: "**sé que existe en la base y quiero reemplazarlo**".

```csharp
class Moto : Vehiculo
{
    public override void Acelerar()
    {
        Velocidad += 15;
    }
}

class Camion : Vehiculo
{
    public override void Acelerar()
    {
        Velocidad += 2;
    }
}

class Sedan : Vehiculo { }    // no sobrescribe: usa la versión de la base
```

La clave está en lo que pasa cuando la variable es del **tipo base**:

```csharp
Vehiculo[] flota = { new Moto(), new Camion(), new Sedan() };

foreach (Vehiculo v in flota)
{
    v.Acelerar();                   // ¿cuál se ejecuta?
    Console.WriteLine(v.Velocidad);
}
// 15
// 2
// 5
```

Aunque las tres variables son `Vehiculo`, cada una ejecuta **la versión de su objeto real**. Eso se llama **despacho dinámico** y es la base del polimorfismo, que se estudia en [Polimorfismo y casting](09-Polimorfismo%20y%20casting.md).

### Extender en lugar de reemplazar: `base.Metodo()`

Un `override` puede llamar a la versión de la base y agregarle algo:

```csharp
class MotoDeportiva : Moto
{
    public override void Acelerar()
    {
        base.Acelerar();            // ejecuta la versión de Moto (+15)
        Console.WriteLine("¡Turbo!");
    }
}
```

### Reglas de `override`

* Solo se puede sobrescribir un miembro `virtual`, `abstract` u `override` de la base (CS0506).
* La firma y el acceso deben coincidir con los de la base: si es `public virtual` en la base, es `public override` en la derivada (CS0507).
* Sirve para métodos y también para **propiedades**:

```csharp
class Vehiculo
{
    public virtual int Ruedas => 4;
}

class Camion : Vehiculo
{
    public override int Ruedas => 8;
}
```

* Un `override` sigue siendo virtual para los niveles siguientes. Para impedir que se siga sobrescribiendo, usa `sealed override`:

```csharp
class Moto : Vehiculo
{
    public sealed override void Acelerar() => Velocidad += 15;   // las derivadas de Moto ya no pueden cambiarlo
}
```

### `override` frente a `new` (ocultación)

Si en la derivada declaras un método con el mismo nombre **sin** `override`, no lo sobrescribes: lo **ocultas**. El compilador te avisa (CS0108) y el comportamiento es muy distinto:

```csharp
class Base
{
    public virtual string Hola() => "Base";
}

class ConOverride : Base
{
    public override string Hola() => "Override";
}

class ConNew : Base
{
    public new string Hola() => "New";
}
```

```csharp
Base a = new ConOverride();
Base b = new ConNew();

Console.WriteLine(a.Hola());               // Override → decide el OBJETO real
Console.WriteLine(b.Hola());               // Base     → decide el tipo de la VARIABLE
Console.WriteLine(((ConNew)b).Hola());     // New
```

En la práctica, casi siempre quieres `override`.

### Métodos abstractos

Un método `abstract` **declara** una operación sin implementarla. Termina en `;` en lugar de llevar un cuerpo:

```csharp
abstract class Vehiculo
{
    public abstract string Describir();     // sin cuerpo: cada derivada DEBE implementarlo
}
```

Es como si la base dijera: "**si heredas de mí, tienes que definir `Describir()`, porque yo no puedo darte una versión por defecto**".

```csharp
class Sedan : Vehiculo
{
    public override string Describir() => "Sedán familiar";   // implementación obligatoria
}

class Camion : Vehiculo { }   // error CS0534: 'Camion' no implementa el miembro abstracto heredado 'Vehiculo.Describir()'
```

Para implementarlo también se usa `override`.

### Clases abstractas

Si una clase tiene algún miembro abstracto, la clase entera **debe** ser abstracta: ¿qué devolvería `new Vehiculo().Describir()` si no hay implementación?

```csharp
class Vehiculo                        // sin abstract
{
    public abstract string Describir();   // error CS0513: 'Vehiculo.Describir()' es abstracto pero está contenido en la clase no abstracta 'Vehiculo'
}
```

Una clase abstracta:

* **No se puede instanciar**: `new Vehiculo()` da `error CS0144`.
* **Puede** tener campos, propiedades, constructores, métodos concretos y métodos virtuales.
* **Puede** no tener ningún miembro abstracto: a veces se marca abstracta solo para impedir que se instancie la base.
* Sus constructores se usan desde las derivadas con `: base(...)`; por eso se suelen declarar `protected`.

```csharp
abstract class Vehiculo
{
    public string Patente { get; }
    public double Velocidad { get; protected set; }

    protected Vehiculo(string patente) => Patente = patente;   // solo lo usan las derivadas

    public abstract string Describir();                          // obligatorio
    public virtual void Acelerar() => Velocidad += 5;            // opcional de sobrescribir
    public void Frenar() => Velocidad = 0;                       // fijo: no se puede sobrescribir
}
```

Los tres tipos de método en una sola clase:

| Tipo | ¿Tiene implementación en la base? | ¿La derivada debe sobrescribirlo? | ¿Puede sobrescribirlo? |
| --- | --- | --- | --- |
| `abstract` | No | **Sí** | Sí |
| `virtual` | Sí | No | Sí |
| Normal (ni `virtual` ni `abstract`) | Sí | No | **No** (solo ocultarlo con `new`) |

### El patrón Template Method

Combinar un método concreto con pasos abstractos permite que la base defina el **algoritmo** y las derivadas, los **detalles**:

```csharp
abstract class Reporte
{
    public string Generar()                     // el "esqueleto", igual para todos
    {
        return $"{Encabezado()}\n{Cuerpo()}\n-- fin --";
    }

    protected virtual string Encabezado() => "REPORTE";   // paso con valor por defecto
    protected abstract string Cuerpo();                    // paso obligatorio
}

class ReporteVentas : Reporte
{
    protected override string Cuerpo() => "Ventas del mes: 120";
}

class ReporteStock : Reporte
{
    protected override string Encabezado() => "INVENTARIO";
    protected override string Cuerpo() => "Productos sin stock: 3";
}
```

-----

## Ejemplo completo

```csharp
Figura[] figuras =
{
    new Circulo(2),
    new Rectangulo(3, 4),
    new Cuadrado(5)
};

foreach (Figura f in figuras)
{
    Console.WriteLine(f.Describir());
}

double areaTotal = figuras.Sum(f => f.Area());
Console.WriteLine($"Área total: {areaTotal:F2}");

abstract class Figura
{
    public string Nombre { get; }

    protected Figura(string nombre) => Nombre = nombre;

    public abstract double Area();                  // cada figura la calcula distinto
    public abstract double Perimetro();

    public virtual string Describir() =>            // descripción por defecto, se puede extender
        $"{Nombre}: área {Area():F2}, perímetro {Perimetro():F2}";
}

class Circulo : Figura
{
    public double Radio { get; }
    public Circulo(double radio) : base("Círculo") => Radio = radio;

    public override double Area() => Math.PI * Radio * Radio;
    public override double Perimetro() => 2 * Math.PI * Radio;
}

class Rectangulo : Figura
{
    public double Ancho { get; }
    public double Alto { get; }

    public Rectangulo(double ancho, double alto) : this("Rectángulo", ancho, alto) { }
    protected Rectangulo(string nombre, double ancho, double alto) : base(nombre)
    {
        Ancho = ancho;
        Alto = alto;
    }

    public override double Area() => Ancho * Alto;
    public override double Perimetro() => 2 * (Ancho + Alto);
}

class Cuadrado : Rectangulo
{
    public Cuadrado(double lado) : base("Cuadrado", lado, lado) { }

    public override string Describir() => $"{base.Describir()} (lado {Ancho})";
}
```

Salida:

```text
Círculo: área 12.57, perímetro 12.57
Rectángulo: área 12.00, perímetro 14.00
Cuadrado: área 25.00, perímetro 20.00 (lado 5)
Área total: 49.57
```

`Figura` no se puede instanciar (¿cuál sería el área de "una figura"?), obliga a implementar `Area` y `Perimetro`, y ofrece una `Describir` virtual que `Cuadrado` extiende con `base.Describir()`. El bucle no sabe qué figura tiene cada posición, y no le hace falta.

-----

## Errores comunes

**1. Sobrescribir un método que no es virtual.**
Qué pasa: `error CS0506: 'Moto.Acelerar()': cannot override inherited member 'Vehiculo.Acelerar()' because it is not marked virtual, abstract, or override`.
Por qué: la base no permitió el reemplazo.
Arreglo: marca el método de la base como `virtual` (si tiene sentido que se reemplace).

**2. Cambiar el acceso al sobrescribir.**
Qué pasa: `error CS0507: 'Moto.Acelerar()': cannot change access modifiers when overriding 'public' inherited member 'Vehiculo.Acelerar()'`.
Por qué: el `override` debe tener el mismo acceso que el miembro original.
Arreglo: usa el mismo modificador.

**3. No implementar un miembro abstracto.**
Qué pasa: `error CS0534: 'Camion' does not implement inherited abstract member 'Vehiculo.Describir()'`.
Por qué: una clase concreta debe implementar todos los miembros abstractos heredados.
Arreglo: agrega `public override string Describir() => ...;` (o marca la derivada como `abstract`).

**4. Miembro abstracto en una clase no abstracta.**
Qué pasa: `error CS0513: 'Vehiculo.Describir()' is abstract but it is contained in non-abstract type 'Vehiculo'`.
Por qué: una clase instanciable no puede tener métodos sin implementación.
Arreglo: `abstract class Vehiculo`.

**5. Instanciar una clase abstracta.**
Qué pasa: `error CS0144: Cannot create an instance of the abstract type or interface 'Figura'`.
Por qué: está incompleta por definición.
Arreglo: instancia una clase derivada concreta.

**6. Darle cuerpo a un método abstracto.**
Qué pasa: `error CS0500: 'Figura.Area()' cannot declare a body because it is marked abstract`.
Por qué: `abstract` significa "sin implementación".
Arreglo: quita el cuerpo y termina con `;`, o cámbialo a `virtual` si quieres una implementación por defecto.

**7. Olvidar `override` y ocultar sin querer.**
Qué pasa: `warning CS0114: 'Moto.Acelerar()' hides inherited member 'Vehiculo.Acelerar()'. To make the current member override that implementation, add the override keyword.` Y el método de la derivada no se ejecuta cuando la variable es de tipo base.
Por qué: sin `override`, el método nuevo oculta al virtual.
Arreglo: agrega `override`.

-----

## Según la versión de C#

* **C# 1:** `virtual`, `override`, `abstract`, `sealed` y `new`.
* **C# 9:** **tipos de retorno covariantes**: un `override` puede devolver un tipo más específico que el de la base (`public override Moto Clonar()` donde la base declara `public virtual Vehiculo Clonar()`).
* **C# 8 / 11:** las interfaces también pueden tener miembros con implementación y miembros `static abstract`, lo que acerca interfaces y clases abstractas (se compara en [Interfaces](08-Interfaces.md)).

-----

## Cuándo sí y cuándo no

**Usa `virtual` cuando:**

* Existe un comportamiento por defecto razonable que algunas derivadas querrán cambiar.

**Usa `abstract` cuando:**

* No existe un comportamiento por defecto razonable y **toda** derivada debe definir el suyo.

**Usa una clase abstracta cuando:**

* Las derivadas comparten **estado** (campos) y **código** concreto, además del contrato.
* Quieres definir un algoritmo con pasos personalizables (Template Method).

**Evita:**

* Marcar todo `virtual` "por si acaso": cada miembro virtual es un punto de extensión que tienes que mantener y documentar.
* Llamar a métodos virtuales desde un constructor: se ejecuta la versión de la derivada antes de que su constructor haya terminado.

-----

## Resumen en 5 líneas

1. `virtual` en la base + `override` en la derivada = reemplazar un comportamiento, respetado aunque la variable sea del tipo base.
2. `base.Metodo()` dentro de un `override` reutiliza la versión de la base.
3. `abstract` = sin implementación; toda derivada concreta debe implementarlo con `override`.
4. Una clase `abstract` no se instancia; puede tener estado, constructores y métodos concretos.
5. Sin `override`, un método con el mismo nombre **oculta** (comportamiento según el tipo de la variable).

-----

## Para profundizar

<details>
<summary>Cómo funciona el despacho dinámico: la tabla virtual</summary>

Cada clase con miembros virtuales tiene una **tabla de métodos virtuales** (vtable): una lista de punteros a las implementaciones. Cada objeto sabe de qué clase es, así que una llamada virtual consulta la tabla del tipo real y salta a la implementación correcta. Una llamada no virtual, en cambio, se resuelve al compilar. El costo extra de una llamada virtual es mínimo, pero impide algunas optimizaciones (como la expansión en línea); `sealed` permite al JIT "desvirtualizar" la llamada.

</details>

<details>
<summary>Sobrescribir los métodos de object</summary>

`ToString()`, `Equals()` y `GetHashCode()` son métodos `virtual` de `object`, la base de todas las clases. Por eso puedes escribir `public override string ToString()` en cualquier clase. Se ve en [La clase Object](10-La%20clase%20Object.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un método `virtual` tiene implementación en la clase base y las derivadas pueden reemplazarlo con `override`. Un método `abstract` no tiene implementación y las derivadas están obligadas a implementarlo. Una clase abstracta no se puede instanciar: sirve como base común, puede tener métodos con y sin implementación, y es obligatoria si la clase tiene algún miembro abstracto.

### Respuesta ampliada (semi-senior)

`virtual`/`override` habilitan el despacho dinámico mediante la vtable: la implementación se elige según el tipo en tiempo de ejecución, a diferencia de la ocultación con `new`, que se resuelve por el tipo estático. Los miembros abstractos son implícitamente virtuales y fuerzan la implementación en las clases concretas. Una clase abstracta puede tener estado, constructores (normalmente `protected`) y lógica compartida, lo que la diferencia de una interfaz; es la base del patrón Template Method. `sealed override` corta la cadena de sobrescritura y habilita la desvirtualización. Desde C# 9, los `override` admiten tipos de retorno covariantes.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `virtual` y `abstract`?**
`virtual` tiene una implementación por defecto y sobrescribirlo es opcional. `abstract` no tiene implementación y sobrescribirlo es obligatorio.

**2. ¿Una clase abstracta puede tener constructor?**
Sí. No se usa con `new` directamente, sino desde las derivadas con `: base(...)`.

**3. ¿Una clase abstracta tiene que tener métodos abstractos?**
No. Puede no tener ninguno; se marca abstracta para impedir que se instancie.

**4. ¿Diferencia entre `override` y `new`?**
`override` reemplaza la implementación para cualquier referencia al objeto (despacho por tipo real). `new` oculta: la versión que se ejecuta depende del tipo de la variable.

-----

## Práctica

**Ejercicio 1.** Crea una clase abstracta `Empleado` con `Nombre`, un método abstracto `CalcularPago()` y un método virtual `Describir()` que devuelva `"<Nombre> cobra <pago>"`. Implementa `EmpleadoFijo` (sueldo mensual fijo) y `EmpleadoPorHora` (tarifa × horas). Recorre un array de `Empleado` y muestra la descripción de cada uno.

<details>
<summary>Solución</summary>

```csharp
Empleado[] equipo =
{
    new EmpleadoFijo("Ana", 4000m),
    new EmpleadoPorHora("Luis", 25m, 120)
};

foreach (var e in equipo)
{
    Console.WriteLine(e.Describir());
}
// Ana cobra 4,000.00
// Luis cobra 3,000.00

abstract class Empleado
{
    public string Nombre { get; }
    protected Empleado(string nombre) => Nombre = nombre;

    public abstract decimal CalcularPago();
    public virtual string Describir() => $"{Nombre} cobra {CalcularPago():N2}";
}

class EmpleadoFijo : Empleado
{
    private readonly decimal _sueldo;
    public EmpleadoFijo(string nombre, decimal sueldo) : base(nombre) => _sueldo = sueldo;
    public override decimal CalcularPago() => _sueldo;
}

class EmpleadoPorHora : Empleado
{
    private readonly decimal _tarifa;
    private readonly int _horas;

    public EmpleadoPorHora(string nombre, decimal tarifa, int horas) : base(nombre)
    {
        _tarifa = tarifa;
        _horas = horas;
    }

    public override decimal CalcularPago() => _tarifa * _horas;
}
```

</details>

**Ejercicio 2.** ¿Qué imprime este código?

```csharp
A x = new C();
x.M1();
x.M2();

class A
{
    public virtual void M1() => Console.WriteLine("A.M1");
    public void M2() => Console.WriteLine("A.M2");
}

class B : A
{
    public override void M1() => Console.WriteLine("B.M1");
}

class C : B
{
    public new void M2() => Console.WriteLine("C.M2");
}
```

<details>
<summary>Solución</summary>

```text
B.M1
A.M2
```

`M1` es virtual: se ejecuta la versión más derivada que lo sobrescribe, que es la de `B` (`C` no lo sobrescribe). `M2` no es virtual y `C` solo lo oculta con `new`; como la variable es de tipo `A`, se ejecuta `A.M2`.

</details>

-----

## Siguiente lección

[Interfaces](08-Interfaces.md)
