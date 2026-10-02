# Value objects

## En una frase

Un **value object** (objeto de valor) es un objeto **sin identidad**, **inmutable** y que se compara **por su contenido**: dos billetes de 10 soles son intercambiables, y un email válido es solo eso, un email válido.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una entidad y la igualdad por identidad: [Entidades](02-Entidades.md).
* Records, `with` e igualdad por valor: [Structs y records](../../01-csharp-core-and-runtime/05-tipos-avanzados/02-Structs%20y%20records.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Value object:** objeto definido solo por sus valores, sin identidad, inmutable.
* **Igualdad estructural (por valor):** dos objetos son iguales si todos sus valores son iguales.
* **Inmutable:** que no cambia después de crearse; para "modificarlo" se crea otro.
* **Obsesión por los primitivos (*primitive obsession*):** usar `string`, `int` o `decimal` para conceptos del negocio que tienen reglas propias.
* **Auto-validación:** el objeto valida sus datos al crearse; si existe, es válido.

-----

## El problema

Mira esta firma:

```csharp
public void RegistrarPago(string email, decimal monto, string moneda, string cuentaOrigen, string cuentaDestino)
```

Y esta llamada, que **compila sin errores**:

```csharp
RegistrarPago("PEN", 150m, "ana@correo.com", "001-222", "001-111");
```

El email y la moneda están intercambiados, y nada lo detecta porque para el compilador todo es `string`. Además:

* ¿Quién valida que el email tenga `@`? ¿Cada método que lo recibe? Entonces la validación está repetida en veinte lugares (o, más probablemente, falta en diez).
* ¿Qué pasa si sumas 100 soles con 100 dólares? Para `decimal` es 200.
* ¿Un monto negativo es válido? Depende de quién escribió cada método.

Esto se llama **obsesión por los primitivos**: conceptos del negocio con reglas (email, dinero, cuenta) representados con tipos genéricos que no saben nada de esas reglas.

-----

## Cómo funciona

### 1. Un tipo para cada concepto, que se valida a sí mismo

```csharp
public sealed record Email
{
    public string Valor { get; }

    private Email(string valor) => Valor = valor;

    public static Email Crear(string? valor)
    {
        if (string.IsNullOrWhiteSpace(valor))
            throw new ArgumentException("El email es obligatorio.", nameof(valor));

        var normalizado = valor.Trim().ToLowerInvariant();
        var arroba = normalizado.IndexOf('@');
        if (arroba <= 0 || arroba != normalizado.LastIndexOf('@') || !normalizado[arroba..].Contains('.'))
            throw new ArgumentException($"'{valor}' no es un email válido.", nameof(valor));

        return new Email(normalizado);
    }

    public override string ToString() => Valor;
}
```

* **Constructor privado + `Crear`:** la única forma de obtener un `Email` es pasar la validación. Si tienes un `Email`, es válido. Ningún método tiene que volver a validarlo.
* **Valida `null` primero:** `valor.Contains("@")` con `null` lanzaría `NullReferenceException` antes de llegar a tu mensaje.
* **Normaliza:** `Ana@Correo.com` y `ana@correo.com ` son el mismo email; al normalizar, la igualdad funciona como espera el negocio.
* **Sin setters:** es inmutable. Para "cambiar" el email de un cliente, se crea **otro** `Email` y se reemplaza.

Ahora la firma se documenta sola y los errores de orden no compilan:

```csharp
public void RegistrarPago(Email email, Dinero monto, NumeroCuenta origen, NumeroCuenta destino)
```

### 2. Igualdad por valor: gratis con `record`

```text
     class sin Equals                         record (value object)
  ┌──────────────────┐ ┌──────────────────┐   ┌──────────────────┐ ┌──────────────────┐
  │ 100 PEN          │ │ 100 PEN          │   │ 100 PEN          │ │ 100 PEN          │
  └──────────────────┘ └──────────────────┘   └──────────────────┘ └──────────────────┘
        a == b → False (dos objetos)                a == b → True (mismos valores)
```

Con una `class` común, dos montos de 100 PEN son **distintos** salvo que sobrescribas `Equals`, `GetHashCode`, `==` y `!=`. Un `record` genera todo eso por ti, comparando cada propiedad:

```csharp
var a = Email.Crear("ana@correo.com");
var b = Email.Crear("  ANA@correo.com ");
Console.WriteLine(a == b);   // True: mismo valor normalizado
```

### 3. Comportamiento: los value objects no son solo datos

Un value object es el lugar natural para las operaciones de ese concepto. El dinero sabe sumarse, pero solo con la misma moneda:

```csharp
public readonly record struct Dinero
{
    public decimal Monto { get; }
    public string Moneda { get; }

    public Dinero(decimal monto, string moneda)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(monto);
        if (string.IsNullOrWhiteSpace(moneda) || moneda.Trim().Length != 3)
            throw new ArgumentException("La moneda debe ser un código ISO de 3 letras (PEN, USD...).", nameof(moneda));
        Monto = decimal.Round(monto, 2);
        Moneda = moneda.Trim().ToUpperInvariant();
    }

    public static Dinero Cero(string moneda) => new(0, moneda);

    public static Dinero operator +(Dinero a, Dinero b)
    {
        ExigirMismaMoneda(a, b);
        return new Dinero(a.Monto + b.Monto, a.Moneda);
    }

    public static Dinero operator *(Dinero a, int cantidad) => new(a.Monto * cantidad, a.Moneda);

    private static void ExigirMismaMoneda(Dinero a, Dinero b)
    {
        if (a.Moneda != b.Moneda)
            throw new InvalidOperationException($"No se puede operar {a.Moneda} con {b.Moneda}.");
    }

    public override string ToString() => $"{Monto:N2} {Moneda}";
}
```

Cada operación devuelve un **nuevo** `Dinero`: el original no cambia. Esto hace que los value objects sean seguros para compartir entre objetos y entre hilos.

### 4. `record`, `record struct` o `class`

| Opción | Cuándo |
| --- | --- |
| `sealed record` (clase) | La opción por defecto. Puede ser `null`, así que distingue "sin valor". |
| `readonly record struct` | Valores pequeños y muy usados (dinero, coordenadas, rangos). Sin asignaciones en el heap. **Cuidado:** `default(Dinero)` existe y salta el constructor (monto 0, moneda `null`). |
| `class` con `Equals`/`GetHashCode`/`==` escritos a mano | Código anterior a C# 9, o cuando necesitas excluir propiedades de la igualdad. |

### 5. ¿Entidad o value object?

La misma "cosa" puede ser una u otra según el dominio:

| Concepto | Como value object | Como entidad |
| --- | --- | --- |
| Dirección | En una tienda: la dirección de envío es solo un valor | En una empresa de correos: cada dirección tiene su historial y su Id |
| Asiento | En un cine sin numeración: "un asiento" | En un avión: el asiento 12A es único y se reserva |
| Línea de pedido | Si nunca se modifica sola: producto + precio + cantidad | Si se edita, se rastrea o se referencia por separado |

La pregunta clave es: **¿me importa cuál es, o solo cómo es?**

-----

## Ejemplo completo

```csharp
var email1 = Email.Crear("Ana@Correo.com");
var email2 = Email.Crear("  ana@correo.com ");
Console.WriteLine($"{email1} == {email2}: {email1 == email2}");

var teclado = new Dinero(150m, "pen");
var mouse = new Dinero(60m, "PEN");
var total = teclado * 2 + mouse;
Console.WriteLine($"Total: {total}");
Console.WriteLine($"teclado sigue valiendo: {teclado}");
Console.WriteLine($"¿Iguales? {new Dinero(10, "USD") == new Dinero(10.00m, "usd")}");

Intentar("Email sin arroba", () => Email.Crear("ana.correo.com"));
Intentar("Email null", () => Email.Crear(null));
Intentar("Monto negativo", () => new Dinero(-5, "PEN"));
Intentar("Sumar PEN con USD", () => teclado + new Dinero(10, "USD"));

static void Intentar(string accion, Func<object> operacion)
{
    try { Console.WriteLine($"✔ {accion}: {operacion()}"); }
    catch (Exception ex) when (ex is ArgumentException or InvalidOperationException)
    {
        Console.WriteLine($"✘ {accion}: {ex.GetType().Name}");
    }
}

sealed record Email
{
    public string Valor { get; }

    private Email(string valor) => Valor = valor;

    public static Email Crear(string? valor)
    {
        if (string.IsNullOrWhiteSpace(valor))
            throw new ArgumentException("El email es obligatorio.", nameof(valor));

        var normalizado = valor.Trim().ToLowerInvariant();
        var arroba = normalizado.IndexOf('@');
        if (arroba <= 0 || arroba != normalizado.LastIndexOf('@') || !normalizado[arroba..].Contains('.'))
            throw new ArgumentException($"'{valor}' no es un email válido.", nameof(valor));

        return new Email(normalizado);
    }

    public override string ToString() => Valor;
}

readonly record struct Dinero
{
    public decimal Monto { get; }
    public string Moneda { get; }

    public Dinero(decimal monto, string moneda)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(monto);
        if (string.IsNullOrWhiteSpace(moneda) || moneda.Trim().Length != 3)
            throw new ArgumentException("La moneda debe ser un código ISO de 3 letras.", nameof(moneda));
        Monto = decimal.Round(monto, 2);
        Moneda = moneda.Trim().ToUpperInvariant();
    }

    public static Dinero operator +(Dinero a, Dinero b)
    {
        if (a.Moneda != b.Moneda)
            throw new InvalidOperationException($"No se puede operar {a.Moneda} con {b.Moneda}.");
        return new Dinero(a.Monto + b.Monto, a.Moneda);
    }

    public static Dinero operator *(Dinero a, int cantidad) => new(a.Monto * cantidad, a.Moneda);

    public override string ToString() => $"{Monto:N2} {Moneda}";
}
```

Salida (con cultura `en-US`; en `es-PE` cambia el separador de miles):

```text
ana@correo.com == ana@correo.com: True
Total: 360.00 PEN
teclado sigue valiendo: 150.00 PEN
¿Iguales? True
✘ Email sin arroba: ArgumentException
✘ Email null: ArgumentException
✘ Monto negativo: ArgumentOutOfRangeException
✘ Sumar PEN con USD: InvalidOperationException
```

`10` y `10.00m` son iguales como `decimal`, y `"usd"` se normaliza a `"USD"`; por eso los dos `Dinero` son iguales.

-----

## Errores comunes

**1. Value object como `class` sin igualdad.**
Qué pasa: `new Dinero(100, "PEN") == new Dinero(100, "PEN")` da `False`.
Por qué: en una clase, `==` y `Equals` comparan referencias por defecto.
Arreglo: usa `record`, o sobrescribe `Equals`, `GetHashCode`, `==` y `!=`.

**2. `Equals` sin `GetHashCode`.**
Qué pasa: advertencia CS0659; en un `HashSet` o como clave de `Dictionary` los valores iguales se duplican o no se encuentran.
Por qué: las colecciones hash usan primero el hash.
Arreglo: sobrescribe los dos, o usa `record`.

**3. Setters públicos (`{ get; set; }`).**
Qué pasa: alguien cambia el monto de un `Dinero` compartido por dos pedidos y los dos cambian.
Por qué: el objeto es mutable y se comparte por referencia.
Arreglo: solo `get`, y operaciones que devuelven una instancia nueva.

**4. Record posicional con `init`: `with` salta la validación.**
Qué pasa: `email with { Valor = "basura" }` crea un email inválido.
Por qué: un record posicional (`record Email(string Valor)`) genera propiedades con `init`, y `with` las asigna sin pasar por tu validación.
Arreglo: propiedades solo con `get` y constructor privado o validado, como en el ejemplo.

**5. Validar después de usar el valor.**
Qué pasa: `valor.Contains('@')` con `valor == null` lanza `NullReferenceException`.
Por qué: el chequeo de nulo se hizo después, o no se hizo.
Arreglo: primero `string.IsNullOrWhiteSpace`, después el resto.

**6. Un record con una `List<T>` adentro.**
Qué pasa: dos records con listas de contenido idéntico son distintos.
Por qué: la igualdad del record compara la lista por **referencia**.
Arreglo: evita colecciones en value objects o implementa la igualdad a mano (`SequenceEqual`).

-----

## Según la versión de C#

* **C# 9:** `record` (igualdad por valor, `with`, `ToString`), `init`.
* **C# 10:** `record struct` y `readonly record struct`; `sealed override ToString` en records.
* **C# 11:** operadores en interfaces estáticas (*generic math*), útil para value objects numéricos.
* **C# 12:** constructores primarios en clases y structs (cuidado: sus parámetros son mutables y no validan solos).
* **EF Core 8:** *complex types* (`ComplexProperty`), pensados para mapear value objects. Antes se usaban *owned types* (`OwnsOne`).

-----

## Cuándo sí y cuándo no

**Usa un value object cuando:**

* Un primitivo tiene reglas: email, DNI, RUC, teléfono, código postal, porcentaje.
* Varios valores van siempre juntos: monto + moneda, latitud + longitud, desde + hasta.
* Quieres que un error de orden o de tipo no compile.

**No lo uses cuando:**

* El valor no tiene reglas ni comportamiento (un comentario libre, una descripción).
* El concepto tiene identidad y ciclo de vida: es una entidad.
* Envuelves todo "por las dudas": un value object por cada `string` es sobreingeniería.

-----

## Resumen en 5 líneas

1. Un value object no tiene identidad: se define y se compara por sus valores.
2. Es inmutable: para cambiarlo se crea otro.
3. Se valida al crearse (constructor privado + `Crear`): si existe, es válido.
4. Concentra el comportamiento del concepto (sumar dinero, normalizar un email).
5. En C# moderno, `sealed record` o `readonly record struct` con propiedades solo `get`.

-----

## Para profundizar

<details>
<summary>Value objects y Entity Framework Core</summary>

```csharp
// EF Core 8+: complex type, sin tabla propia ni Id
modelBuilder.Entity<Pedido>().ComplexProperty(p => p.Total, d =>
{
    d.Property(x => x.Monto).HasColumnName("TotalMonto");
    d.Property(x => x.Moneda).HasColumnName("TotalMoneda").HasMaxLength(3);
});

// Value object de un solo valor: conversión de valor
modelBuilder.Entity<Cliente>()
    .Property(c => c.Email)
    .HasConversion(e => e.Valor, v => Email.Crear(v));
```

Las columnas quedan en la tabla de la entidad dueña. La configuración vive en infraestructura, no en el value object.

</details>

<details>
<summary>¿Excepciones o un resultado?</summary>

`Email.Crear` lanza una excepción si el dato es inválido. Cuando el dato viene del usuario y es esperable que sea inválido, algunos equipos prefieren devolver un resultado (`Result<Email>` o un `bool TryCrear(string, out Email)`), y reservar las excepciones para errores de programación. Ambos enfoques son válidos; lo importante es que **nunca** exista un `Email` inválido.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un value object es un objeto que no tiene Id y se compara por sus valores, como un email o un monto de dinero. Es inmutable: si quieres otro valor, creas otro objeto. Se valida al crearse, así que si existe, es válido. En C# es muy cómodo hacerlo con `record`, porque ya trae la igualdad por valor.

### Respuesta ampliada (semi-senior)

Los value objects atacan la obsesión por los primitivos: encapsulan validación, normalización y comportamiento de un concepto (`Money` con operaciones que exigen la misma moneda, `Email` normalizado). Son inmutables, lo que los hace seguros para compartir, y tienen igualdad estructural. En C# uso `sealed record` o `readonly record struct` con propiedades solo `get` y una factory, porque un record posicional con `init` permite saltarse la validación con `with`, y un struct tiene un `default` que salta el constructor. En EF Core los mapeo con complex types u owned types, o conversiones de valor cuando son de un solo campo. La misma noción puede ser entidad o value object según el bounded context.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué deben ser inmutables?**
Porque se comparten: si un `Dinero` cambiara, cambiaría en todos los objetos que lo referencian. Además, la igualdad por valor solo es estable si el valor no cambia.

**2. ¿Qué es la obsesión por los primitivos?**
Usar `string`, `int` o `decimal` para conceptos con reglas, repartiendo la validación y permitiendo errores que el compilador no detecta.

**3. ¿Un value object puede tener métodos?**
Sí, y debería: las operaciones del concepto (`Sumar`, `Convertir`, `EsMayorQue`) son su lugar natural, siempre devolviendo instancias nuevas.

-----

## Práctica

**Ejercicio 1.** Crea el value object `RangoFechas` con `Desde` y `Hasta` (`DateOnly`). Reglas: `Desde` no puede ser posterior a `Hasta`. Agrega `Dias` (cantidad de días, inclusive) y `SeSolapaCon(RangoFechas otro)`.

<details>
<summary>Solución</summary>

```csharp
var enero = new RangoFechas(new DateOnly(2026, 1, 1), new DateOnly(2026, 1, 31));
var quincena = new RangoFechas(new DateOnly(2026, 1, 20), new DateOnly(2026, 2, 5));
Console.WriteLine(enero.Dias);                     // 31
Console.WriteLine(enero.SeSolapaCon(quincena));    // True

sealed record RangoFechas
{
    public DateOnly Desde { get; }
    public DateOnly Hasta { get; }

    public RangoFechas(DateOnly desde, DateOnly hasta)
    {
        if (desde > hasta)
            throw new ArgumentException("'Desde' no puede ser posterior a 'Hasta'.", nameof(desde));
        (Desde, Hasta) = (desde, hasta);
    }

    public int Dias => Hasta.DayNumber - Desde.DayNumber + 1;

    public bool SeSolapaCon(RangoFechas otro) => Desde <= otro.Hasta && otro.Desde <= Hasta;
}
```

</details>

**Ejercicio 2.** Esta clase está en las notas de clase. ¿Qué problemas tiene?

```csharp
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}

var m1 = new Money { Amount = 100, Currency = "USD" };
var m2 = new Money { Amount = 100, Currency = "USD" };
Console.WriteLine(m1 == m2);
```

<details>
<summary>Solución</summary>

1. Imprime **`False`**: es una clase sin igualdad por valor, así que `==` compara referencias. No cumple la definición de value object.
2. Es mutable (`set`): no es un value object.
3. No valida nada: monto negativo, moneda vacía o `null` (además, `Currency` sin inicializar da la advertencia CS8618).
4. No tiene comportamiento: sumar dos `Money` de distinta moneda queda en manos de quien lo use.

Versión corregida: el `Dinero` de esta lección.

</details>

-----

## Siguiente lección

[Agregados](04-Agregados.md)
