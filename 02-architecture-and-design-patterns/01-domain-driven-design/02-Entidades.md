# Entidades

## En una frase

Una **entidad** es un objeto del dominio que se distingue por su **identidad** (un `Id`) y no por sus datos: un cliente sigue siendo **el mismo cliente** aunque cambie de nombre, de email y de dirección.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un modelo rico y el lenguaje ubicuo: [Modelo anémico y modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md).
* `Equals`, `GetHashCode` y `==`: [Structs y records](../../01-csharp-core-and-runtime/05-tipos-avanzados/02-Structs%20y%20records.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Entidad:** objeto con identidad propia que perdura y cambia a lo largo del tiempo.
* **Identidad:** el dato que distingue a una entidad de todas las demás, normalmente un `Id`.
* **Ciclo de vida:** los estados por los que pasa una entidad desde que se crea hasta que se archiva o elimina.
* **Igualdad por identidad:** dos objetos representan la misma entidad si tienen el mismo `Id`, aunque sus demás datos difieran.
* **Método de fábrica (*factory*):** método estático que crea un objeto válido, en lugar de un constructor público.

-----

## El problema

Un cliente, Ana, cambia su email. En el sistema aparecen dos objetos cargados en momentos distintos:

```csharp
var antes   = new Cliente { Id = 7, Nombre = "Ana", Email = "ana@viejo.com" };
var despues = new Cliente { Id = 7, Nombre = "Ana", Email = "ana@nuevo.com" };
```

¿Son el mismo cliente? Para el negocio, **sí**: es Ana, con el mismo historial de compras. Si los comparas por sus datos, dirías que son clientes distintos y podrías duplicarla, perder su historial o enviarle un descuento dos veces.

Al revés: dos clientes que se llaman "Juan Pérez" y viven en la misma calle **no** son la misma persona. Los datos iguales no implican la misma entidad.

Lo que define a un cliente no son sus datos, que cambian, sino **su identidad**, que no cambia nunca.

-----

## Cómo funciona

### 1. La identidad es inmutable; el resto puede cambiar

```csharp
public class Cliente
{
    public Guid Id { get; }                       // solo get: nunca cambia
    public string Nombre { get; private set; }    // cambia, pero solo a través de métodos
    public string Email { get; private set; }

    public Cliente(Guid id, string nombre, string email)
    {
        if (id == Guid.Empty) throw new ArgumentException("El Id es obligatorio.", nameof(id));
        Id = id;
        CambiarNombre(nombre);
        CambiarEmail(email);
    }

    public void CambiarNombre(string nombre)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(nombre);
        Nombre = nombre.Trim();
    }

    public void CambiarEmail(string email)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(email);
        if (!email.Contains('@')) throw new ArgumentException("Email inválido.", nameof(email));
        Email = email.Trim();
    }
}
```

* `Id` solo tiene `get`: se asigna al crear y queda fijo para siempre.
* Los demás datos tienen `private set` y cambian solo con métodos que validan. El constructor **reutiliza** esos métodos, así la validación vive en un solo lugar.
* Una entidad nunca existe en un estado inválido: si el constructor termina, el objeto cumple sus reglas.

(En la lección siguiente, el email deja de ser un `string` y pasa a ser un value object: [Value objects](03-Value%20objects.md).)

### 2. La igualdad es por identidad

Por defecto, `Equals` y `==` en una clase comparan **referencias** (¿es el mismo objeto en memoria?). Para una entidad, lo correcto es comparar el `Id`:

```text
           Igualdad por referencia          Igualdad por identidad
           (por defecto en una clase)       (lo que quieres en una entidad)

  a ──►  ┌────────────┐                  a ──►  ┌────────────┐
         │ Id = 7     │                         │ Id = 7     │
         │ Ana, viejo │                         │ Ana, viejo │
         └────────────┘                         └────────────┘
  b ──►  ┌────────────┐                  b ──►  ┌────────────┐
         │ Id = 7     │                         │ Id = 7     │
         │ Ana, nuevo │                         │ Ana, nuevo │
         └────────────┘                         └────────────┘
  a.Equals(b) → false (objetos distintos)  a.Equals(b) → true (mismo Id)
```

Una clase base reutilizable:

```csharp
public abstract class Entidad : IEquatable<Entidad>
{
    public Guid Id { get; }

    protected Entidad(Guid id)
    {
        if (id == Guid.Empty) throw new ArgumentException("El Id es obligatorio.", nameof(id));
        Id = id;
    }

    public bool Equals(Entidad? otra) =>
        otra is not null && otra.GetType() == GetType() && otra.Id == Id;

    public override bool Equals(object? obj) => Equals(obj as Entidad);
    public override int GetHashCode() => HashCode.Combine(GetType(), Id);

    public static bool operator ==(Entidad? a, Entidad? b) => a is null ? b is null : a.Equals(b);
    public static bool operator !=(Entidad? a, Entidad? b) => !(a == b);
}
```

Detalles importantes:

* **Siempre sobrescribe `GetHashCode` junto con `Equals`.** Si no, el compilador avisa (CS0659) y los diccionarios y `HashSet` se comportan mal: dos clientes "iguales" quedarían en buckets distintos.
* **Compara también el tipo:** un `Cliente` y un `Producto` con el mismo `Guid` no son iguales.
* **Sobrescribe `==` y `!=`**, o `a == b` seguirá comparando referencias aunque `Equals` diga otra cosa.

### 3. ¿Quién genera el Id?

| Estrategia | Ventaja | Desventaja |
| --- | --- | --- |
| `Guid.NewGuid()` / `Guid.CreateVersion7()` en el código | La entidad tiene Id **desde que se crea**, sin ir a la base de datos | Ocupa 16 bytes; el Guid v4 es aleatorio y fragmenta índices |
| Autoincremental de la base de datos (`int IDENTITY`) | Compacto y legible | La entidad no tiene Id hasta que se guarda: igualdad y eventos antes de guardar se complican |

En DDD suele preferirse generar el Id en el dominio. Desde **.NET 9**, `Guid.CreateVersion7()` genera Guids ordenados por tiempo, que no fragmentan los índices.

### 4. Métodos de fábrica para crear con intención

Un constructor público con muchos parámetros no dice **qué** está pasando. Un método de fábrica sí:

```csharp
public class Cliente : Entidad
{
    public string Nombre { get; private set; }
    public DateTime RegistradoEn { get; }

    private Cliente(Guid id, string nombre, DateTime registradoEn) : base(id)
    {
        Nombre = nombre;
        RegistradoEn = registradoEn;
    }

    public static Cliente Registrar(string nombre)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(nombre);
        return new Cliente(Guid.CreateVersion7(), nombre.Trim(), DateTime.UtcNow);
    }
}
```

`Cliente.Registrar("Ana")` habla el lenguaje del negocio ("se registra un cliente") y el constructor privado garantiza que no haya otra forma de crearlo.

-----

## Ejemplo completo

```csharp
var ana = Cliente.Registrar("Ana", "ana@viejo.com");
var mismaAnaDesdeLaBase = Cliente.Reconstituir(ana.Id, "Ana", "ana@nuevo.com");
var otraAna = Cliente.Registrar("Ana", "ana@viejo.com");

Console.WriteLine($"ana == mismaAnaDesdeLaBase: {ana == mismaAnaDesdeLaBase}");  // mismo Id
Console.WriteLine($"ana == otraAna: {ana == otraAna}");                            // mismos datos, otro Id

var clientes = new HashSet<Cliente> { ana, mismaAnaDesdeLaBase, otraAna };
Console.WriteLine($"Clientes distintos en el conjunto: {clientes.Count}");

ana.CambiarNombre("  Ana María ");
Console.WriteLine($"Nuevo nombre: '{ana.Nombre}' (Id sin cambios: {ana.Id == mismaAnaDesdeLaBase.Id})");

try { ana.CambiarNombre(" "); }
catch (ArgumentException ex) { Console.WriteLine($"Rechazado: {ex.ParamName}"); }

abstract class Entidad : IEquatable<Entidad>
{
    public Guid Id { get; }

    protected Entidad(Guid id)
    {
        if (id == Guid.Empty) throw new ArgumentException("El Id es obligatorio.", nameof(id));
        Id = id;
    }

    public bool Equals(Entidad? otra) =>
        otra is not null && otra.GetType() == GetType() && otra.Id == Id;

    public override bool Equals(object? obj) => Equals(obj as Entidad);
    public override int GetHashCode() => HashCode.Combine(GetType(), Id);

    public static bool operator ==(Entidad? a, Entidad? b) => a is null ? b is null : a.Equals(b);
    public static bool operator !=(Entidad? a, Entidad? b) => !(a == b);
}

class Cliente : Entidad
{
    public string Nombre { get; private set; } = "";
    public string Email { get; private set; } = "";

    private Cliente(Guid id, string nombre, string email) : base(id)
    {
        CambiarNombre(nombre);
        CambiarEmail(email);
    }

    // Crear un cliente nuevo: genera la identidad.
    public static Cliente Registrar(string nombre, string email) =>
        new(Guid.CreateVersion7(), nombre, email);

    // Recrear un cliente que ya existe (por ejemplo, leído de la base de datos): conserva la identidad.
    public static Cliente Reconstituir(Guid id, string nombre, string email) =>
        new(id, nombre, email);

    public void CambiarNombre(string nombre)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(nombre);
        Nombre = nombre.Trim();
    }

    public void CambiarEmail(string email)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(email);
        if (!email.Contains('@')) throw new ArgumentException("Email inválido.", nameof(email));
        Email = email.Trim();
    }
}
```

Salida:

```text
ana == mismaAnaDesdeLaBase: True
ana == otraAna: False
Clientes distintos en el conjunto: 2
Nuevo nombre: 'Ana María' (Id sin cambios: True)
Rechazado: nombre
```

El `HashSet` tiene 2 elementos, no 3: `ana` y `mismaAnaDesdeLaBase` son la misma entidad, aunque tengan emails distintos.

-----

## Errores comunes

**1. Sobrescribir `Equals` sin `GetHashCode`.**
Qué pasa: advertencia CS0659; en un `HashSet` o `Dictionary` dos entidades "iguales" no se encuentran o se duplican.
Por qué: las colecciones hash primero comparan el hash y solo después llaman a `Equals`.
Arreglo: sobrescribe siempre los dos, con el mismo criterio (`HashCode.Combine(GetType(), Id)`).

**2. Olvidar `==`.**
Qué pasa: `a.Equals(b)` da `true` pero `a == b` da `false`.
Por qué: en una clase, `==` compara referencias salvo que lo sobrecargues.
Arreglo: sobrecarga `==` y `!=` en la clase base, delegando en `Equals`.

**3. Id con `set` público.**
Qué pasa: alguien cambia el `Id` y la entidad "se convierte" en otra; los diccionarios que la contenían quedan corruptos.
Por qué: el hash cambia mientras el objeto está guardado.
Arreglo: `Id` solo con `get`, asignado en el constructor.

**4. Usar `record` para una entidad.**
Qué pasa: dos instancias con el mismo `Id` pero distinto email son **distintas** (igualdad por todos los valores), y `with` crea "otra" entidad con el mismo Id.
Por qué: los records están pensados para igualdad por valor, justo lo contrario de una entidad.
Arreglo: `class` con igualdad por identidad. Los records son para value objects.

**5. Validar solo en el constructor.**
Qué pasa: se crea válida, pero `CambiarNombre("")` la deja inválida.
Por qué: la validación quedó duplicada o se olvidó en el método.
Arreglo: un único método que valida y asigna, usado también por el constructor.

**6. `throw new Exception("...")` genérico.**
Qué pasa: quien llama no puede distinguir un error de validación de un fallo inesperado sin capturar todo.
Por qué: `Exception` no comunica la categoría del error.
Arreglo: `ArgumentException` (dato inválido), `InvalidOperationException` (operación no permitida en el estado actual) o una excepción de dominio propia (`DomainException`).

-----

## Según la versión de C#

* **C# 7.3 / .NET Core 2.1:** `HashCode.Combine`, para calcular el hash sin fórmulas manuales.
* **C# 8:** tipos de referencia que aceptan null (`Entidad?`) en las firmas de `Equals`.
* **C# 9:** `is not null`; patrones lógicos.
* **.NET 6 / 7 / 8:** `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegativeOrZero`.
* **.NET 9:** `Guid.CreateVersion7()`, Guids ordenados por tiempo.

-----

## Cuándo sí y cuándo no

**Es una entidad cuando:**

* Te importa **cuál** es, no solo **cómo** es: cliente, pedido, cuenta, factura, alumno.
* Tiene un ciclo de vida (se crea, cambia de estado, se archiva).
* Necesitas seguirla en el tiempo o referenciarla desde otros objetos.

**No es una entidad (probablemente es un value object) cuando:**

* Dos instancias con los mismos datos son intercambiables: un monto de dinero, un email, una dirección, un rango de fechas.
* No tiene sentido "cambiarla": si cambia, es otro valor.

-----

## Resumen en 5 líneas

1. Una entidad se define por su identidad, no por sus datos.
2. El `Id` es inmutable; los demás datos cambian solo con métodos que validan.
3. La igualdad es por `Id` (y tipo): sobrescribe `Equals`, `GetHashCode`, `==` y `!=`.
4. Crea las entidades con constructores privados y métodos de fábrica con nombres del negocio.
5. Las entidades son `class`, no `record`.

-----

## Para profundizar

<details>
<summary>¿Y Entity Framework?</summary>

EF Core puede mapear entidades con setters privados, constructores privados y campos de respaldo, así que el dominio no tiene que "abrirse" para la base de datos:

* EF usa un constructor privado sin parámetros, o uno cuyos parámetros coincidan con propiedades.
* Las propiedades con `private set` se mapean sin configuración extra.
* Las colecciones privadas (`_lineas`) se mapean con `HasMany(...).WithOne()` y `UsePropertyAccessMode(PropertyAccessMode.Field)`.

Ese mapeo vive en la capa de infraestructura (`IEntityTypeConfiguration<T>`), no en la clase de dominio.

</details>

<details>
<summary>¿Comparar también por tipo con proxies de EF?</summary>

Con *lazy loading*, EF crea subclases proxy (`Castle.Proxies.ClienteProxy`), y `GetType()` no coincide con `Cliente`. Si usas proxies, compara con un tipo "no proxy" (por ejemplo, `NHibernateUtil.GetClass` en NHibernate o `ObjectContext.GetObjectType` en EF6) o evita los proxies, que es lo más común en EF Core.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una entidad es un objeto que tiene identidad propia, normalmente un Id, y que puede cambiar sus datos con el tiempo sin dejar de ser la misma. Por ejemplo, un cliente que cambia de email sigue siendo el mismo cliente. Por eso dos entidades se comparan por su Id, no por sus propiedades.

### Respuesta ampliada (semi-senior)

Una entidad tiene continuidad e identidad a lo largo de su ciclo de vida. Su Id es inmutable y la igualdad se implementa por identidad y tipo, sobrescribiendo `Equals`, `GetHashCode` y los operadores, normalmente en una clase base `Entity`. Su estado solo cambia con métodos que expresan operaciones del negocio y validan invariantes; los setters son privados y la construcción se hace con factories que garantizan un estado inicial válido. Prefiero generar el Id en el dominio (por ejemplo, Guid v7) para no depender de la base de datos. No uso `record` para entidades porque su igualdad estructural contradice la identidad.

### Preguntas frecuentes de seguimiento

**1. ¿Cuál es la diferencia entre una entidad y un value object?**
La entidad tiene identidad y cambia en el tiempo; el value object no tiene identidad, es inmutable y se compara por sus valores.

**2. ¿Por qué hay que sobrescribir `GetHashCode` si sobrescribes `Equals`?**
Porque las colecciones hash usan el hash para ubicar los elementos; objetos iguales deben tener el mismo hash.

**3. ¿Por qué no usar un `record` como entidad?**
Porque compara por todos sus valores: dos versiones de la misma entidad serían distintas.

-----

## Práctica

**Ejercicio 1.** Crea la entidad `Producto` (hereda de `Entidad`) con `Nombre` y `Precio`. Reglas: el nombre no puede estar vacío; el precio debe ser mayor que cero; `CambiarPrecio` no permite aumentar más del 50% de una vez. Créalo con `Producto.Crear(nombre, precio)`.

<details>
<summary>Solución</summary>

```csharp
var p = Producto.Crear("Teclado", 100m);
p.CambiarPrecio(140m);
Console.WriteLine(p.Precio);   // 140

try { p.CambiarPrecio(500m); }
catch (InvalidOperationException ex) { Console.WriteLine(ex.Message); }

class Producto : Entidad
{
    public string Nombre { get; private set; } = "";
    public decimal Precio { get; private set; }

    private Producto(Guid id, string nombre, decimal precio) : base(id)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(nombre);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(precio);
        Nombre = nombre.Trim();
        Precio = precio;
    }

    public static Producto Crear(string nombre, decimal precio) =>
        new(Guid.CreateVersion7(), nombre, precio);

    public void CambiarPrecio(decimal nuevo)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(nuevo);
        if (nuevo > Precio * 1.5m)
            throw new InvalidOperationException("No se puede aumentar el precio más de un 50% de una vez.");
        Precio = nuevo;
    }
}

// + la clase Entidad de la lección
```

</details>

**Ejercicio 2.** ¿Qué imprime este código y por qué? ¿Cómo lo arreglas?

```csharp
class Cliente
{
    public int Id { get; set; }
    public override bool Equals(object? obj) => obj is Cliente c && c.Id == Id;
}

var set = new HashSet<Cliente> { new() { Id = 1 }, new() { Id = 1 } };
Console.WriteLine(set.Count);
```

<details>
<summary>Solución</summary>

Lo más probable es que imprima **2**. `GetHashCode` no está sobrescrito (advertencia CS0659), así que cada objeto tiene un hash basado en su referencia; el `HashSet` los pone en buckets distintos y nunca llega a llamar a `Equals`.

Arreglo: `public override int GetHashCode() => Id.GetHashCode();` y, además, `Id` sin `set` público (si el Id cambia mientras el objeto está en el conjunto, queda "perdido").

</details>

-----

## Siguiente lección

[Value objects](03-Value%20objects.md)
