# Modificadores de acceso y encapsulación

## En una frase

Los modificadores de acceso (`public`, `private`, `protected`, `internal` y sus combinaciones) deciden **desde dónde** se puede usar cada miembro; la **encapsulación** consiste en ocultar el estado interno de un objeto (`private`) y exponer solo operaciones que mantienen ese estado siempre válido.

-----

## Antes de empezar

Conviene que ya sepas:

* Definir clases con campos y métodos, de [Clases y objetos](01-Clases%20y%20objetos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Modificador de acceso:** palabra clave que define la visibilidad de un tipo o miembro.
* **Encapsulación:** ocultar los detalles internos de un objeto y exponer una interfaz pública controlada.
* **Invariante:** regla que el estado de un objeto debe cumplir siempre (por ejemplo, "el saldo nunca es negativo").
* **Ensamblado (*assembly*):** el `.dll` o `.exe` que produce un proyecto. Es la unidad que usa `internal`.
* **Clase derivada:** clase que hereda de otra (se ve en [Herencia](06-Herencia.md)).
* **`readonly`:** modificador de campo que solo permite asignarlo en la declaración o en el constructor.
* **API pública:** el conjunto de miembros que otros pueden usar.

-----

## El problema

En la lección anterior, la cuenta bancaria tenía este agujero:

```csharp
var cuenta = new CuentaBancaria { Titular = "Ana" };
cuenta.Depositar(100);
cuenta.Saldo = -1_000_000;    // ¡nada lo impide!
```

El método `Retirar` validaba que el saldo no quedara negativo, pero cualquier código podía **saltarse esa validación** escribiendo directamente en el campo. Si `Saldo` es público, la regla "el saldo nunca es negativo" depende de que **todos** los programadores, en todos los archivos, se acuerden de respetarla. Y eso no va a pasar.

Necesitas que la propia clase **impida** el uso incorrecto.

-----

## Cómo funciona

### `public` y `private`

```csharp
class CuentaBancaria
{
    private decimal _saldo;            // solo accesible DENTRO de CuentaBancaria
    public string Titular = "";        // accesible desde cualquier lugar

    public void Depositar(decimal monto)
    {
        if (monto <= 0) return;
        _saldo += monto;               // ✅ dentro de la clase se puede
    }

    public bool Retirar(decimal monto)
    {
        if (monto <= 0 || monto > _saldo) return false;
        _saldo -= monto;
        return true;
    }

    public decimal ConsultarSaldo() => _saldo;
}
```

```csharp
var cuenta = new CuentaBancaria { Titular = "Ana" };
cuenta.Depositar(100);
Console.WriteLine(cuenta.ConsultarSaldo());   // 100

cuenta._saldo = -1_000_000;   // error CS0122: '_saldo' is inaccessible due to its protection level
```

Ahora la única forma de cambiar el saldo es a través de `Depositar` y `Retirar`, que validan. La regla del saldo vive en **un solo lugar** y nadie puede saltársela.

* `private`: solo el código **dentro de la misma clase** puede usarlo.
* `public`: cualquier código puede usarlo.
* Convención: los campos privados se nombran `_camelCase` (`_saldo`).

### El acceso por defecto

Si no escribes ningún modificador:

| Elemento | Acceso por defecto |
| --- | --- |
| Miembros de una clase o struct (campos, métodos...) | `private` |
| Clases y structs de nivel superior (no anidados) | `internal` |
| Tipos anidados dentro de otra clase | `private` |
| Miembros de una interfaz | `public` |
| Valores de un `enum` | `public` |

C# elige **el acceso más restrictivo razonable** por defecto: lo que no expones explícitamente, queda oculto. Ojo: una clase sin modificador **no** es `public`, es `internal`. Igualmente, la buena práctica es **escribir siempre el modificador**, para que la intención quede clara.

### Todos los modificadores de acceso

| Modificador | Accesible desde |
| --- | --- |
| `public` | Cualquier lugar. |
| `private` | Solo la misma clase (o struct). |
| `protected` | La misma clase y sus clases derivadas. |
| `internal` | Cualquier código del mismo ensamblado (proyecto). |
| `protected internal` | El mismo ensamblado **o** las clases derivadas de otros ensamblados. |
| `private protected` | Las clases derivadas **que además están** en el mismo ensamblado. |
| `file` | Solo el archivo `.cs` donde se declara. **Solo se aplica a tipos**, no a miembros. |

Visualmente, de más abierto a más cerrado:

```text
public  ─►  protected internal  ─►  internal / protected  ─►  private protected  ─►  private
(todos)                                                                              (solo la clase)
```

En el día a día usarás sobre todo `public`, `private` y, cuando haya herencia, `protected`. `internal` aparece en librerías y en soluciones con varios proyectos, para ocultar detalles a los demás proyectos.

### Qué es un ensamblado y por qué importa `internal`

Cada proyecto (`.csproj`) se compila en un ensamblado (`.dll`). En una solución típica:

```text
MiTienda.sln
├── MiTienda.Dominio      → MiTienda.Dominio.dll
├── MiTienda.Datos        → MiTienda.Datos.dll
└── MiTienda.Api          → MiTienda.Api.dll   (referencia a los otros dos)
```

Un tipo `internal` en `MiTienda.Datos` se puede usar en cualquier parte de ese proyecto, pero **no** desde `MiTienda.Api`. Así, cada proyecto expone solo lo que los demás deben usar.

### Encapsulación: más que poner `private`

La encapsulación es una forma de **programación defensiva**: proteges el funcionamiento interno de una clase para que otro código no pueda romperla. Implica:

1. **Ocultar el estado:** los campos son `private`.
2. **Exponer operaciones con significado:** `Depositar`, `Retirar`, no "cambiar el número del saldo".
3. **Proteger las invariantes:** cada operación pública deja al objeto en un estado válido.
4. **Esconder la implementación:** si mañana el saldo se calcula a partir de un historial de movimientos, el código que usa `ConsultarSaldo()` no se entera.

Beneficios:

* **Control:** decides qué se puede hacer y qué no.
* **Mantenimiento:** puedes cambiar el interior sin romper a quienes usan la clase.
* **Depuración:** si el saldo es incorrecto, el error está en uno de los pocos métodos que lo modifican.

### `readonly`: datos que no cambian después de crear el objeto

```csharp
class CuentaBancaria
{
    private readonly string _numero;    // se asigna una vez y no cambia más

    public CuentaBancaria(string numero)   // constructor: se ve en una lección próxima
    {
        _numero = numero;
    }

    public void CambiarNumero(string nuevo)
    {
        _numero = nuevo;   // error CS0191: un campo readonly no se puede asignar (excepto en un constructor)
    }
}
```

`readonly` protege los datos **también de la propia clase**: es una garantía más fuerte que `private`.

### Getters y setters como métodos

Antes de conocer las propiedades, el patrón para exponer un campo privado de forma controlada son dos métodos:

```csharp
private int _edad;

public int ObtenerEdad() => _edad;

public void EstablecerEdad(int valor)
{
    if (valor < 0) throw new ArgumentOutOfRangeException(nameof(valor), "La edad no puede ser negativa.");
    _edad = valor;
}
```

Funciona, pero es verboso. C# tiene una sintaxis específica para esto, las **propiedades**, que son el tema de la [próxima lección](03-Propiedades.md).

-----

## Ejemplo completo

```csharp
var cuenta = new CuentaBancaria("Ana", "001-123");

cuenta.Depositar(500);
cuenta.Retirar(120);
bool ok = cuenta.Retirar(10_000);

Console.WriteLine(cuenta.Resumen());
Console.WriteLine($"Retiro grande permitido: {ok}");
Console.WriteLine("Movimientos:");
foreach (string m in cuenta.ObtenerMovimientos())
{
    Console.WriteLine($"  {m}");
}

public class CuentaBancaria
{
    private readonly string _titular;
    private readonly string _numero;
    private decimal _saldo;
    private readonly List<string> _movimientos = new();

    public CuentaBancaria(string titular, string numero)
    {
        _titular = titular;
        _numero = numero;
    }

    public void Depositar(decimal monto)
    {
        ValidarMonto(monto);
        _saldo += monto;
        Registrar($"+{monto:N2}");
    }

    public bool Retirar(decimal monto)
    {
        ValidarMonto(monto);
        if (monto > _saldo)
        {
            Registrar($"Rechazado -{monto:N2}");
            return false;
        }

        _saldo -= monto;
        Registrar($"-{monto:N2}");
        return true;
    }

    public string Resumen() => $"{_titular} ({_numero}): {_saldo:N2}";

    // Se devuelve una copia: quien llama no puede modificar la lista interna
    public string[] ObtenerMovimientos() => _movimientos.ToArray();

    // Detalles de implementación: privados
    private static void ValidarMonto(decimal monto)
    {
        if (monto <= 0)
            throw new ArgumentOutOfRangeException(nameof(monto), "El monto debe ser positivo.");
    }

    private void Registrar(string texto) => _movimientos.Add($"{DateTime.Now:HH:mm:ss} {texto}");
}
```

Salida (las horas varían):

```text
Ana (001-123): 380.00
Retiro grande permitido: False
Movimientos:
  10:15:02 +500.00
  10:15:02 -120.00
  10:15:02 Rechazado -10,000.00
```

Fíjate en `ObtenerMovimientos()`: devuelve una **copia** del historial. Si devolviera la lista interna, cualquiera podría hacer `cuenta.ObtenerMovimientos().Clear()` y borrarlo. Encapsular también es cuidar las referencias que dejas escapar.

-----

## Errores comunes

**1. Acceder a un miembro privado desde fuera.**
Qué pasa: `error CS0122: 'CuentaBancaria._saldo' is inaccessible due to its protection level`.
Por qué: `private` solo permite el acceso dentro de la clase.
Arreglo: usa el método o la propiedad pública que la clase ofrece. Si no existe, pregúntate si debería existir antes de hacer público el campo.

**2. Creer que una clase sin modificador es pública.**
Qué pasa: otro proyecto no puede usar la clase: `error CS0122` (o CS0246 si ni siquiera la encuentra).
Por qué: el acceso por defecto de un tipo de nivel superior es `internal`.
Arreglo: `public class ...` si debe usarse desde otros ensamblados.

**3. Exponer un tipo más privado que el miembro que lo usa.**
Qué pasa: `error CS0050: Inconsistent accessibility: return type 'Movimiento' is less accessible than method 'CuentaBancaria.UltimoMovimiento()'`.
Por qué: un método `public` no puede devolver un tipo `internal` o `private`, porque quien lo llama no podría usarlo.
Arreglo: haz el tipo igual de accesible o reduce la visibilidad del método.

**4. Usar `file` en un miembro.**
Qué pasa: `error CS0106: The modifier 'file' is not valid for this item`.
Por qué: `file` solo se aplica a tipos (clases, structs, interfaces...).
Arreglo: usa `private` para miembros.

**5. Devolver la colección interna.**
Qué pasa: no hay error de compilación, pero el código externo modifica el estado del objeto sin pasar por sus reglas.
Por qué: devolver la lista entrega una referencia a ella.
Arreglo: devuelve una copia (`ToArray()`) o una vista de solo lectura (`AsReadOnly()`, `IReadOnlyList<T>`).

-----

## Según la versión de C#

* **C# 1:** `public`, `private`, `protected`, `internal` y `protected internal`.
* **C# 7.2:** `private protected`.
* **C# 11:** `file` para tipos visibles solo en su archivo (muy usado por generadores de código).

-----

## Cuándo sí y cuándo no

**Regla general: empieza por lo más restrictivo** y abre solo lo necesario.

| Elemento | Acceso recomendado |
| --- | --- |
| Campos | `private` (siempre) |
| Métodos auxiliares internos | `private` |
| Operaciones que la clase ofrece | `public` |
| Miembros pensados para clases hijas | `protected` |
| Clases de implementación dentro de una librería | `internal` |

**Evita:**

* Campos `public` en clases con reglas: usa propiedades o métodos.
* Hacer `public` algo "por si acaso": todo lo público es un compromiso que otros pueden empezar a usar y que después cuesta cambiar.

-----

## Resumen en 5 líneas

1. `public`: desde cualquier lugar; `private`: solo dentro de la clase; `protected`: la clase y sus hijas; `internal`: el mismo proyecto.
2. Por defecto, los miembros son `private` y las clases de nivel superior son `internal` (no `public`).
3. Encapsular = campos privados + operaciones públicas que mantienen las invariantes.
4. `readonly` impide cambiar un campo después del constructor, incluso desde la propia clase.
5. No dejes escapar referencias a colecciones internas: devuelve copias o vistas de solo lectura.

-----

## Para profundizar

<details>
<summary>private es por clase, no por objeto</summary>

`private` restringe el acceso por **tipo**, no por instancia. Un método de `CuentaBancaria` puede leer el `_saldo` privado de **otra** cuenta:

```csharp
public bool TieneMasSaldoQue(CuentaBancaria otra) => _saldo > otra._saldo;   // ✅ compila
```

Es útil para comparaciones, igualdad y copias entre objetos del mismo tipo.

</details>

<details>
<summary>InternalsVisibleTo: abrir internal a las pruebas</summary>

Si un proyecto de pruebas necesita probar clases `internal`, el proyecto original puede abrirle el acceso:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="MiTienda.Datos.Tests" />
</ItemGroup>
```

Úsalo con moderación: lo ideal es probar a través de la API pública.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los modificadores de acceso controlan quién puede usar un miembro: `public` desde cualquier lugar, `private` solo dentro de la clase, `protected` la clase y sus derivadas, e `internal` el mismo proyecto. La encapsulación consiste en dejar los datos privados y exponer métodos o propiedades públicas que controlan cómo se modifican, para que el objeto siempre esté en un estado válido.

### Respuesta ampliada (semi-senior)

Hay siete niveles: `public`, `protected internal` (unión: ensamblado o derivadas), `internal`, `protected`, `private protected` (intersección: derivadas dentro del ensamblado), `private` y `file` (solo para tipos). Por defecto, los miembros son `private` y los tipos de nivel superior, `internal`. La encapsulación no es solo ocultar campos: es proteger las invariantes del objeto, exponer operaciones con significado de negocio en lugar de setters genéricos y no filtrar referencias mutables (colecciones internas) por la API. `readonly` refuerza la inmutabilidad dentro de la propia clase. En soluciones con varios proyectos, `internal` delimita qué parte de un ensamblado es API y qué parte es implementación.

### Preguntas frecuentes de seguimiento

**1. ¿Cuál es el acceso por defecto de una clase?**
`internal` si es de nivel superior; `private` si está anidada dentro de otra clase.

**2. ¿Diferencia entre `protected internal` y `private protected`?**
`protected internal` permite el acceso desde el mismo ensamblado **o** desde derivadas en cualquier ensamblado. `private protected` lo permite solo a las derivadas que **también** están en el mismo ensamblado.

**3. ¿Encapsulación es lo mismo que poner getters y setters?**
No. Un getter y un setter públicos sin reglas exponen el campo igual que si fuera público. Encapsular es exponer operaciones que protegen las reglas del objeto.

-----

## Práctica

**Ejercicio 1.** Encapsula esta clase para que la temperatura nunca baje de -273.15 (el cero absoluto) y solo se pueda modificar con un método `Ajustar(double delta)`:

```csharp
class Termometro
{
    public double Temperatura;
}
```

<details>
<summary>Solución</summary>

```csharp
class Termometro
{
    private const double CeroAbsoluto = -273.15;
    private double _temperatura;

    public double Leer() => _temperatura;

    public bool Ajustar(double delta)
    {
        double nueva = _temperatura + delta;
        if (nueva < CeroAbsoluto) return false;

        _temperatura = nueva;
        return true;
    }
}
```

</details>

**Ejercicio 2.** Indica si cada línea compila (las tres clases están en el mismo proyecto):

```csharp
class A
{
    private int x;
    protected int y;
    internal int z;
    public int w;
}

class B : A          // B hereda de A
{
    void Probar()
    {
        x = 1;   // (1)
        y = 1;   // (2)
        z = 1;   // (3)
    }
}

class C
{
    void Probar(A a)
    {
        a.y = 1; // (4)
        a.z = 1; // (5)
        a.w = 1; // (6)
    }
}
```

<details>
<summary>Solución</summary>

1. No: `x` es `private` de `A` (CS0122).
2. Sí: `protected` es accesible desde una clase derivada.
3. Sí: `internal` y el mismo proyecto.
4. No: `C` no deriva de `A` (CS0122).
5. Sí: mismo proyecto.
6. Sí: `public`.

</details>

-----

## Siguiente lección

[Propiedades](03-Propiedades.md)
