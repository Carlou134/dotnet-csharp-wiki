# Ejercicios de DDD: entidades, value objects y agregados

Cuaderno de práctica basado en los ejercicios de la Sesión 8, **revisados y corregidos**. Mantienen los nombres en inglés de la clase (`Customer`, `Email`, `Order`) para que los reconozcas. Cada solución es un programa completo con *top-level statements*.

Requisitos previos: [Entidades](02-Entidades.md), [Value objects](03-Value%20objects.md) y [Agregados](04-Agregados.md).

-----

## Ejercicios guiados

### Ejercicio 1: entidad `Customer`

**Objetivo:** crear una entidad con identidad inmutable.

**Contexto:** un cliente se identifica por su Id; su nombre puede cambiar.

**Instrucciones:**

1. Crea `Customer` con `Id` (`Guid`) y `Name`.
2. El `Id` se asigna al crear y no cambia nunca.
3. `Name` no se puede asignar desde afuera.

<details>
<summary>Solución</summary>

```csharp
var customer = new Customer(Guid.NewGuid(), "Ana");
Console.WriteLine($"{customer.Id} - {customer.Name}");

// customer.Id = Guid.NewGuid();   // CS0200: la propiedad es de solo lectura
// customer.Name = "Otro";         // CS0272: el set es inaccesible

class Customer
{
    public Guid Id { get; }
    public string Name { get; private set; }

    public Customer(Guid id, string name)
    {
        if (id == Guid.Empty) throw new ArgumentException("El Id es obligatorio.", nameof(id));
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        Id = id;
        Name = name.Trim();
    }
}
```

</details>

**Qué observar:** `Id { get; }` (sin `set`) es más fuerte que `{ get; private set; }`: ni siquiera la propia clase puede cambiarlo después del constructor. En la guía original el constructor no validaba nada: un cliente con `Guid.Empty` o nombre vacío era "válido".

-----

### Ejercicio 2: value object `Email`

**Objetivo:** reemplazar un `string` por un tipo que se valida a sí mismo.

**Contexto:** en la guía, `Email` era una clase con `if (!value.Contains("@")) throw new Exception(...)`.

**Instrucciones:**

1. Crea `Email` inmutable, con igualdad por valor.
2. Rechaza `null`, vacío y textos sin un único `@` con algo antes y un `.` después.
3. Normaliza (sin espacios, en minúsculas).

<details>
<summary>Solución</summary>

```csharp
var a = new Email("Ana@Correo.com ");
var b = new Email("ana@correo.com");
Console.WriteLine($"{a} == {b}: {a == b}");   // True

foreach (var texto in new string?[] { null, "", "ana.correo.com", "@correo.com", "ana@correo" })
{
    try { _ = new Email(texto); }
    catch (ArgumentException ex) { Console.WriteLine($"'{texto}': {ex.Message}"); }
}

sealed record Email
{
    public string Value { get; }

    public Email(string? value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("El email es obligatorio.", nameof(value));

        var normalized = value.Trim().ToLowerInvariant();
        var at = normalized.IndexOf('@');
        if (at <= 0 || at != normalized.LastIndexOf('@') || !normalized[at..].Contains('.'))
            throw new ArgumentException("Formato de email inválido.", nameof(value));

        Value = normalized;
    }

    public override string ToString() => Value;
}
```

</details>

**Qué observar:**

* En la guía, `value.Contains("@")` con `null` lanzaba `NullReferenceException` en lugar del error de validación. Primero se valida el nulo.
* `throw new Exception` genérico no deja distinguir un dato inválido de un fallo real: `ArgumentException`.
* "Contiene `@`" acepta `"@"` o `"a@@b"`. La validación del ejercicio es razonable pero simple; un email solo se confirma de verdad enviando un correo.
* Es un `record` con propiedad solo `get`: igualdad por valor gratis y `with` no puede saltarse la validación.

-----

### Ejercicio 3: `ChangeName` con validación

**Objetivo:** que la entidad solo cambie a través de un método que valida.

**Contexto:** el nombre de un cliente puede cambiar, pero nunca quedar vacío ni superar 100 caracteres.

**Instrucciones:**

1. Agrega `ChangeName(string newName)` a `Customer`.
2. La validación debe estar en **un** solo lugar, compartido con el constructor.

<details>
<summary>Solución</summary>

```csharp
var customer = new Customer(Guid.NewGuid(), "Ana");
customer.ChangeName("  Ana María  ");
Console.WriteLine($"'{customer.Name}'");   // 'Ana María'

try { customer.ChangeName("   "); }
catch (ArgumentException ex) { Console.WriteLine(ex.Message); }

try { customer.ChangeName(new string('x', 101)); }
catch (ArgumentException ex) { Console.WriteLine(ex.Message); }

Console.WriteLine($"'{customer.Name}'");   // sigue 'Ana María': un cambio inválido no deja rastro

class Customer
{
    public const int MaxNameLength = 100;

    public Guid Id { get; }
    public string Name { get; private set; } = "";

    public Customer(Guid id, string name)
    {
        if (id == Guid.Empty) throw new ArgumentException("El Id es obligatorio.", nameof(id));
        Id = id;
        ChangeName(name);
    }

    public void ChangeName(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("El nombre es obligatorio.", nameof(newName));

        var trimmed = newName.Trim();
        if (trimmed.Length > MaxNameLength)
            throw new ArgumentException($"El nombre no puede superar {MaxNameLength} caracteres.", nameof(newName));

        Name = trimmed;
    }
}
```

</details>

**Qué observar:** la validación ocurre **antes** de asignar. Si falla, el objeto queda exactamente como estaba. El constructor llama a `ChangeName`, así que la regla no se duplica.

-----

### Ejercicio 4: el agregado `Order`

**Objetivo:** crear una raíz de agregado que controla su colección interna.

**Contexto:** un pedido tiene líneas (`OrderItem`). Desde afuera no se pueden agregar ni quitar líneas directamente.

**Instrucciones:**

1. `Order` con `Id`, `CustomerId` (referencia por Id a otro agregado) y una lista privada de `OrderItem`.
2. Expón las líneas de solo lectura.
3. `OrderItem` con `ProductName`, `UnitPrice` y `Quantity`.

<details>
<summary>Solución</summary>

```csharp
var order = new Order(Guid.NewGuid(), Guid.NewGuid());
Console.WriteLine($"Pedido {order.Id} del cliente {order.CustomerId}: {order.Items.Count} líneas");

// order.Items.Add(...);   // CS1061: IReadOnlyList<OrderItem> no tiene Add

sealed record OrderItem(string ProductName, decimal UnitPrice, int Quantity);

class Order
{
    private readonly List<OrderItem> _items = new();

    public Guid Id { get; }
    public Guid CustomerId { get; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public Order(Guid id, Guid customerId)
    {
        if (id == Guid.Empty) throw new ArgumentException("El Id es obligatorio.", nameof(id));
        if (customerId == Guid.Empty) throw new ArgumentException("El cliente es obligatorio.", nameof(customerId));
        (Id, CustomerId) = (id, customerId);
    }
}
```

</details>

**Qué observar:**

* `CustomerId` y no `Customer`: entre agregados se referencia por Id.
* `_items` es `readonly`: nadie puede reemplazar la lista entera, ni siquiera dentro de la clase.
* La guía llamaba a `OrderItem` "entidad interna", pero no tenía identidad. Tal como está es un **value object** (por eso aquí es un `record`). Si las líneas se editan y se identifican, por ejemplo por producto, pasan a ser entidades locales (como en el ejercicio 8).

-----

### Ejercicio 5: `AddItem` con reglas

**Objetivo:** que agregar una línea pase por las invariantes del agregado.

**Contexto:** reglas del negocio: cantidad entre 1 y 10, precio mayor que cero, sin productos repetidos (si se repite, se suman las cantidades respetando el máximo) y como máximo 20 líneas.

**Instrucciones:** implementa `AddItem(string productName, decimal unitPrice, int quantity)` en `Order`.

<details>
<summary>Solución</summary>

```csharp
var order = new Order(Guid.NewGuid(), Guid.NewGuid());
order.AddItem("Teclado", 150m, 2);
order.AddItem("teclado", 150m, 3);   // mismo producto: se suma
order.AddItem("Mouse", 60m, 1);

foreach (var item in order.Items) Console.WriteLine(item);

try { order.AddItem("Teclado", 150m, 6); }    // 5 + 6 = 11 > 10
catch (InvalidOperationException ex) { Console.WriteLine(ex.Message); }

try { order.AddItem("Cable", 0m, 1); }
catch (ArgumentOutOfRangeException ex) { Console.WriteLine(ex.ParamName); }

sealed record OrderItem(string ProductName, decimal UnitPrice, int Quantity);

class Order
{
    public const int MaxItems = 20;
    public const int MaxQuantityPerItem = 10;

    private readonly List<OrderItem> _items = new();

    public Guid Id { get; }
    public Guid CustomerId { get; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public Order(Guid id, Guid customerId) => (Id, CustomerId) = (id, customerId);

    public void AddItem(string productName, decimal unitPrice, int quantity)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(productName);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(unitPrice);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);

        var index = _items.FindIndex(i =>
            string.Equals(i.ProductName, productName.Trim(), StringComparison.OrdinalIgnoreCase));

        if (index >= 0)
        {
            var existing = _items[index];
            var newQuantity = existing.Quantity + quantity;
            EnsureQuantity(newQuantity);
            _items[index] = existing with { Quantity = newQuantity };   // OrderItem es inmutable: se reemplaza
            return;
        }

        EnsureQuantity(quantity);
        if (_items.Count == MaxItems)
            throw new InvalidOperationException($"Un pedido no puede tener más de {MaxItems} líneas.");

        _items.Add(new OrderItem(productName.Trim(), unitPrice, quantity));
    }

    private static void EnsureQuantity(int quantity)
    {
        if (quantity > MaxQuantityPerItem)
            throw new InvalidOperationException($"Máximo {MaxQuantityPerItem} unidades por producto.");
    }
}
```

Salida:

```text
OrderItem { ProductName = Teclado, UnitPrice = 150, Quantity = 5 }
OrderItem { ProductName = Mouse, UnitPrice = 60, Quantity = 1 }
Máximo 10 unidades por producto.
unitPrice
```

</details>

**Qué observar:**

* Las reglas que dependen de **varias** líneas (repetidos, máximo de líneas) solo puede verificarlas la raíz. Por eso las líneas no se agregan desde afuera.
* Como `OrderItem` es un value object inmutable, "sumar cantidad" es **reemplazar** la línea por otra (`with`).
* ¿Y el precio si el producto ya estaba con otro precio? Es una decisión del negocio: aquí se conserva el original. Pregúntalo antes de decidirlo tú.

-----

### Ejercicio 6: `IReadOnlyCollection` no es suficiente

**Objetivo:** comprobar por qué `IReadOnlyCollection<T> Items => _items;` no protege la colección.

**Contexto:** la guía usaba `public IReadOnlyCollection<OrderItem> Items => _items;`.

**Instrucciones:**

1. Crea dos clases: una expone `=> _items` y otra `=> _items.AsReadOnly()`.
2. Intenta modificar la colección de cada una con un cast a `List<T>` y a `ICollection<T>`.

<details>
<summary>Solución</summary>

```csharp
var insegura = new OrderInsegura();
var segura = new OrderSegura();

Probar("Insegura, cast a List", () => ((List<string>)insegura.Items).Add("línea trucha"));
Console.WriteLine($"  Insegura tiene {insegura.Items.Count} líneas");

Probar("Segura, cast a List", () => ((List<string>)segura.Items).Add("línea trucha"));
Probar("Segura, cast a ICollection", () => ((ICollection<string>)segura.Items).Add("línea trucha"));
Console.WriteLine($"  Segura tiene {segura.Items.Count} líneas");

static void Probar(string caso, Action accion)
{
    try { accion(); Console.WriteLine($"✔ {caso}: se modificó"); }
    catch (Exception ex) { Console.WriteLine($"✘ {caso}: {ex.GetType().Name}"); }
}

class OrderInsegura
{
    private readonly List<string> _items = new() { "Teclado" };
    public IReadOnlyCollection<string> Items => _items;
}

class OrderSegura
{
    private readonly List<string> _items = new() { "Teclado" };
    public IReadOnlyCollection<string> Items => _items.AsReadOnly();
}
```

Salida:

```text
✔ Insegura, cast a List: se modificó
  Insegura tiene 2 líneas
✘ Segura, cast a List: InvalidCastException
✘ Segura, cast a ICollection: NotSupportedException
  Segura tiene 1 líneas
```

</details>

**Qué observar:** una interfaz **oculta** métodos, pero no cambia el objeto: detrás sigue la misma `List<T>`. `AsReadOnly()` devuelve un `ReadOnlyCollection<T>` que **envuelve** la lista (sin copiarla) y rechaza cualquier escritura. Además, refleja los cambios que la raíz hace después en `_items`.

-----

### Ejercicio 7: `Equals` en un value object

**Objetivo:** entender qué hace falta para que una clase compare por valor, y por qué `record` lo resuelve.

**Contexto:** la guía sobrescribía solo `Equals` con `obj as Email`.

**Instrucciones:**

1. Escribe `Money` como **clase** con igualdad por valor completa (`Equals`, `GetHashCode`, `==`, `!=`, `IEquatable<Money>`).
2. Compárala con la versión `record`.

<details>
<summary>Solución</summary>

```csharp
var a = new MoneyClase(100m, "USD");
var b = new MoneyClase(100m, "usd");
Console.WriteLine($"Clase:  Equals={a.Equals(b)}  =={a == b}  HashSet={new HashSet<MoneyClase> { a, b }.Count}");

var c = new MoneyRecord(100m, "USD");
var d = new MoneyRecord(100m, "USD");
Console.WriteLine($"Record: Equals={c.Equals(d)}  =={c == d}  HashSet={new HashSet<MoneyRecord> { c, d }.Count}");

sealed class MoneyClase : IEquatable<MoneyClase>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public MoneyClase(decimal amount, string currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        Amount = amount;
        Currency = currency.Trim().ToUpperInvariant();
    }

    public bool Equals(MoneyClase? other) =>
        other is not null && Amount == other.Amount && Currency == other.Currency;

    public override bool Equals(object? obj) => Equals(obj as MoneyClase);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);

    public static bool operator ==(MoneyClase? x, MoneyClase? y) => x is null ? y is null : x.Equals(y);
    public static bool operator !=(MoneyClase? x, MoneyClase? y) => !(x == y);
}

// Lo mismo, generado por el compilador (sin validación: ver la nota abajo).
sealed record MoneyRecord(decimal Amount, string Currency);
```

Salida:

```text
Clase:  Equals=True  ==True  HashSet=1
Record: Equals=True  ==True  HashSet=1
```

</details>

**Qué observar:**

* La versión de la guía (solo `Equals`) tenía tres problemas:
  1. **Advertencia CS0659** (falta `GetHashCode`): en un `HashSet` o `Dictionary` los valores iguales se duplican.
  2. `==` seguía comparando **referencias**.
  3. La clase no era `sealed`: con `obj as Email`, una subclase puede romper la simetría (`a.Equals(b)` distinto de `b.Equals(a)`).
* El record posicional del final es cómodo, pero **no valida** y sus propiedades son `init` (con `with` se salta cualquier validación). Para un value object real, usa la forma con propiedades solo `get` de la lección [Value objects](03-Value%20objects.md).
* `HashCode.Combine(Amount, Currency)` es seguro con decimales como `100m` y `100.00m`: `decimal.GetHashCode` da el mismo hash para valores iguales con distinta escala.

-----

### Ejercicio 8: caso completo, `Order` con `GetTotal`

**Objetivo:** juntar entidad, value object y agregado.

**Contexto:** pedido con líneas, dinero con moneda, estados y total calculado. En la guía, el primer `Order` hacía `Total += price` sin validar nada.

**Instrucciones:**

1. Value object `Money` (monto ≥ 0, moneda de 3 letras, suma solo con la misma moneda, multiplicación por cantidad).
2. `OrderItem` como entidad local identificada por `ProductId`, cuya cantidad solo cambia desde el pedido (`internal`).
3. `Order` como raíz: `AddItem`, `ChangeQuantity`, `RemoveItem`, `Confirm` y `GetTotal()`.
4. Reglas: moneda única por pedido, no se modifica después de confirmar, no se confirma vacío.

<details>
<summary>Solución</summary>

```csharp
var teclado = Guid.NewGuid();
var mouse = Guid.NewGuid();

var order = Order.Create(Guid.NewGuid(), "PEN");
order.AddItem(teclado, new Money(150m, "PEN"), 2);
order.AddItem(mouse, new Money(60m, "PEN"), 1);
order.ChangeQuantity(mouse, 3);
Console.WriteLine($"Total: {order.GetTotal()}");   // 300 + 180 = 480.00 PEN

order.RemoveItem(mouse);
order.Confirm();
Console.WriteLine($"Confirmado: {order.Status}, total {order.GetTotal()}");

try { order.AddItem(mouse, new Money(60m, "PEN"), 1); }
catch (DomainException ex) { Console.WriteLine(ex.Message); }

try { Order.Create(Guid.NewGuid(), "PEN").Confirm(); }
catch (DomainException ex) { Console.WriteLine(ex.Message); }

class DomainException(string message) : Exception(message);

enum OrderStatus { Draft, Confirmed }

readonly record struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);
        if (string.IsNullOrWhiteSpace(currency) || currency.Trim().Length != 3)
            throw new ArgumentException("La moneda debe tener 3 letras (ISO 4217).", nameof(currency));
        Amount = decimal.Round(amount, 2);
        Currency = currency.Trim().ToUpperInvariant();
    }

    public static Money Zero(string currency) => new(0m, currency);

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException($"No se puede sumar {a.Currency} con {b.Currency}.");
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public static Money operator *(Money a, int quantity) => new(a.Amount * quantity, a.Currency);

    public override string ToString() => $"{Amount:N2} {Currency}";
}

sealed class OrderItem
{
    public Guid ProductId { get; }
    public Money UnitPrice { get; }
    public int Quantity { get; private set; }
    public Money Subtotal => UnitPrice * Quantity;

    internal OrderItem(Guid productId, Money unitPrice, int quantity)
    {
        ProductId = productId;
        UnitPrice = unitPrice;
        ChangeQuantity(quantity);
    }

    internal void ChangeQuantity(int quantity)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);
        Quantity = quantity;
    }
}

sealed class Order
{
    private readonly List<OrderItem> _items = new();

    public Guid Id { get; }
    public Guid CustomerId { get; }
    public string Currency { get; }
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private Order(Guid id, Guid customerId, string currency)
    {
        if (customerId == Guid.Empty) throw new ArgumentException("El cliente es obligatorio.", nameof(customerId));
        Id = id;
        CustomerId = customerId;
        Currency = Money.Zero(currency).Currency;   // reutiliza la validación de Money
    }

    public static Order Create(Guid customerId, string currency) =>
        new(Guid.CreateVersion7(), customerId, currency);

    public void AddItem(Guid productId, Money unitPrice, int quantity)
    {
        EnsureDraft();
        if (unitPrice.Currency != Currency)
            throw new DomainException($"El pedido es en {Currency}.");
        if (_items.Exists(i => i.ProductId == productId))
            throw new DomainException("El producto ya está en el pedido; usa ChangeQuantity.");
        _items.Add(new OrderItem(productId, unitPrice, quantity));
    }

    public void ChangeQuantity(Guid productId, int quantity)
    {
        EnsureDraft();
        Find(productId).ChangeQuantity(quantity);
    }

    public void RemoveItem(Guid productId)
    {
        EnsureDraft();
        _items.Remove(Find(productId));
    }

    public void Confirm()
    {
        EnsureDraft();
        if (_items.Count == 0) throw new DomainException("No se puede confirmar un pedido vacío.");
        Status = OrderStatus.Confirmed;
    }

    public Money GetTotal() =>
        _items.Aggregate(Money.Zero(Currency), (total, item) => total + item.Subtotal);

    private OrderItem Find(Guid productId) =>
        _items.Find(i => i.ProductId == productId)
        ?? throw new DomainException("El producto no está en el pedido.");

    private void EnsureDraft()
    {
        if (Status != OrderStatus.Draft) throw new DomainException("El pedido ya no se puede modificar.");
    }
}
```

Salida:

```text
Total: 480.00 PEN
Confirmado: Confirmed, total 300.00 PEN
El pedido ya no se puede modificar.
No se puede confirmar un pedido vacío.
```

</details>

**Qué observar:**

* `GetTotal()` **calcula**, no acumula. Con `Total += price` (como en la guía), quitar o cambiar una línea dejaba el total desincronizado.
* El total parte de `Money.Zero(Currency)`, no de `default(Money)`: el `default` de un struct salta el constructor y tendría `Currency = null`.
* `OrderItem` aquí **sí** es una entidad local: se identifica por `ProductId` dentro del pedido y su cantidad cambia. Su constructor y `ChangeQuantity` son `internal`: en un proyecto de dominio separado, la API no puede llamarlos.
* Errores de programación (`ArgumentException`) frente a reglas de negocio (`DomainException`): la capa de aplicación puede traducir las segundas a un 400 o 422 con un mensaje para el usuario.

-----

## Retos

### Reto 1: `Address` como value object

**Misión:** modela `Address` (calle, ciudad, código postal y país) como value object y úsalo en `Customer` con `ChangeAddress`. Dos direcciones con los mismos datos, aunque tengan distinto uso de mayúsculas o espacios, deben ser iguales.

**Pista:** normaliza en el constructor; `string.Equals` con `OrdinalIgnoreCase` no se usa en la igualdad de un record, así que normaliza los valores antes de guardarlos.

<details>
<summary>Solución</summary>

```csharp
var a = new Address(" Av. Arequipa 123 ", "Lima", "15046", "pe");
var b = new Address("av. arequipa 123", "LIMA", "15046", "PE");
Console.WriteLine(a == b);   // True

var customer = new Customer(Guid.NewGuid(), "Ana");
customer.ChangeAddress(a);
Console.WriteLine(customer.Address);

sealed record Address
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public Address(string street, string city, string postalCode, string country)
    {
        Street = Normalize(street, nameof(street));
        City = Normalize(city, nameof(city));
        PostalCode = Normalize(postalCode, nameof(postalCode));
        Country = Normalize(country, nameof(country));
        if (Country.Length != 2) throw new ArgumentException("El país es un código ISO de 2 letras.", nameof(country));
    }

    private static string Normalize(string value, string name)
    {
        if (string.IsNullOrWhiteSpace(value)) throw new ArgumentException($"{name} es obligatorio.", name);
        return string.Join(' ', value.Split(' ', StringSplitOptions.RemoveEmptyEntries)).ToUpperInvariant();
    }

    public override string ToString() => $"{Street}, {City} {PostalCode}, {Country}";
}

class Customer(Guid id, string name)
{
    public Guid Id { get; } = id;
    public string Name { get; private set; } = name;
    public Address? Address { get; private set; }

    public void ChangeAddress(Address address) => Address = address ?? throw new ArgumentNullException(nameof(address));
}
```

</details>

### Reto 2: igualdad de entidades con clase base

**Misión:** crea una clase base `Entity` con igualdad por Id y tipo, y demuestra que un `Customer` y un `Product` con el mismo `Guid` **no** son iguales, pero dos `Customer` con el mismo Id y distinto nombre **sí** lo son.

**Pista:** compara `GetType()` además del `Id`, e incluye el tipo en `GetHashCode`.

<details>
<summary>Solución</summary>

```csharp
var id = Guid.NewGuid();
Console.WriteLine(new Customer(id, "Ana") == new Customer(id, "Ana María"));   // True
Console.WriteLine(new Customer(id, "Ana").Equals(new Product(id, "Teclado")));  // False

abstract class Entity(Guid id) : IEquatable<Entity>
{
    public Guid Id { get; } = id;

    public bool Equals(Entity? other) => other is not null && other.GetType() == GetType() && other.Id == Id;
    public override bool Equals(object? obj) => Equals(obj as Entity);
    public override int GetHashCode() => HashCode.Combine(GetType(), Id);

    public static bool operator ==(Entity? a, Entity? b) => a is null ? b is null : a.Equals(b);
    public static bool operator !=(Entity? a, Entity? b) => !(a == b);
}

class Customer(Guid id, string name) : Entity(id)
{
    public string Name { get; private set; } = name;
}

class Product(Guid id, string name) : Entity(id)
{
    public string Name { get; private set; } = name;
}
```

</details>

### Reto 3: invariante con descuento

**Misión:** agrega a `Order` (ejercicio 8) un método `ApplyDiscount(decimal percentage)`. Reglas: entre 0 y 30%, solo en borrador, un único descuento por pedido y el total con descuento nunca es menor que cero. `GetTotal()` debe reflejarlo.

**Pista:** guarda el porcentaje, no el total descontado; `GetTotal()` sigue calculando a partir de las líneas.

<details>
<summary>Solución</summary>

Agrega a `Order`:

```csharp
public decimal DiscountPercentage { get; private set; }

public void ApplyDiscount(decimal percentage)
{
    EnsureDraft();
    if (percentage is <= 0 or > 30)
        throw new DomainException("El descuento debe estar entre 0 y 30%.");
    if (DiscountPercentage > 0)
        throw new DomainException("El pedido ya tiene un descuento.");
    DiscountPercentage = percentage;
}

public Money GetTotal()
{
    var subtotal = _items.Aggregate(Money.Zero(Currency), (total, item) => total + item.Subtotal);
    var discounted = subtotal.Amount * (1 - DiscountPercentage / 100m);
    return new Money(Math.Max(0, discounted), Currency);
}
```

Con 300 PEN y 10%: `270.00 PEN`. Si después se agrega una línea, el descuento se aplica también sobre ella, porque se calcula y no se guarda un total. (Si el negocio dice que el descuento era solo sobre lo que había al aplicarlo, eso es **otra** regla: pregúntala.)

</details>

-----

## Checkpoint

Antes de seguir, deberías poder responder sin mirar:

* ¿Por qué `Customer` se compara por Id y `Email` por valor?
* ¿Qué tres cosas hay que sobrescribir, además de `Equals`, para que una clase compare por valor de verdad?
* ¿Por qué `IReadOnlyCollection<T> Items => _items;` no protege la lista y qué usas en su lugar?
* ¿Por qué `GetTotal()` calcula en lugar de acumular con `Total += ...`?
* ¿Por qué `Order` guarda `CustomerId` y no un `Customer`?
