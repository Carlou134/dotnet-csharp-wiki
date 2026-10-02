# Agregados

## En una frase

Un **agregado** es un grupo de entidades y value objects que **cambian juntos** y se tratan como una unidad; tiene una única puerta de entrada, la **raíz del agregado** (*aggregate root*), que es la responsable de que el grupo completo cumpla siempre sus reglas.

-----

## Antes de empezar

Conviene que ya sepas:

* [Entidades](02-Entidades.md) y [Value objects](03-Value%20objects.md).
* Colecciones de solo lectura e interfaces de colección: [Listas](../../01-csharp-core-and-runtime/06-colecciones/01-Listas.md).
* Interfaces e inyección de dependencias: [Principios DRY KISS y refactoring](../../01-csharp-core-and-runtime/11-codigo-limpio/02-Principios%20DRY%20KISS%20y%20refactoring.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Agregado (*aggregate*):** grupo de objetos del dominio que se modifica y se guarda como una unidad.
* **Raíz del agregado (*aggregate root*):** la entidad principal del agregado; el único objeto al que se accede desde afuera.
* **Frontera del agregado:** qué objetos están dentro (y se protegen juntos) y cuáles quedan afuera.
* **Invariante del agregado:** regla que involucra a varios objetos del grupo ("el pedido no supera 20 líneas", "no hay dos líneas del mismo producto").
* **Consistencia transaccional:** todo lo que está dentro del agregado se guarda en la misma transacción: o todo, o nada.
* **Repositorio:** abstracción que guarda y recupera agregados completos, como si fueran una colección en memoria.

-----

## El problema

Un pedido tiene líneas. Con entidades "sueltas", cualquiera puede tocar cualquier cosa:

```csharp
var linea = lineaRepository.Obtener(lineaId);
linea.Cantidad = 500;               // ¿el pedido tenía un límite de stock por línea?
lineaRepository.Guardar(linea);     // ¿y el total del pedido? ¿y si ya estaba confirmado?

pedido.Lineas.Add(new LineaPedido(...));   // ¿y si ese producto ya estaba en otra línea?
```

Cada regla del pedido ("no se modifica confirmado", "no hay productos repetidos", "máximo 20 líneas") depende de **varias** líneas a la vez. Si las líneas se modifican por su cuenta, nadie puede garantizar esas reglas: cada línea es válida por sí sola, pero el pedido en conjunto queda inconsistente.

-----

## Cómo funciona

### 1. Una frontera y una sola puerta

```text
          ┌───────────────── Agregado Pedido ─────────────────┐
          │                                                   │
 afuera   │   ┌───────────────────────┐                       │
 ───────► │   │  Pedido (RAÍZ)        │──┬──► LineaPedido     │
 solo se  │   │  Id, ClienteId,       │  ├──► LineaPedido     │
 habla    │   │  Estado, Total        │  └──► LineaPedido     │
 con la   │   └───────────────────────┘                       │
 raíz     │        │                                          │
          │        └──► DireccionEnvio (value object)         │
          └───────────────────────────────────────────────────┘
                   │                          │
                   │ ClienteId (solo el Id)   │ ProductoId (solo el Id)
                   ▼                          ▼
          ┌─────────────────┐        ┌─────────────────┐
          │ Agregado Cliente│        │ Agregado Producto│
          └─────────────────┘        └─────────────────┘
```

Las reglas de un agregado:

1. **Desde afuera solo se referencia a la raíz.** Nadie obtiene una `LineaPedido` para modificarla directamente; se le pide al `Pedido`: `pedido.CambiarCantidad(productoId, 3)`.
2. **La raíz protege las invariantes de todo el grupo.** Antes de aceptar un cambio, comprueba que el agregado completo siga siendo válido.
3. **Entre agregados, se referencia por Id.** El pedido guarda `ClienteId`, no un objeto `Cliente`. Así un agregado no puede modificar otro "de paso".
4. **Una transacción modifica un solo agregado.** Lo que está adentro se guarda junto (consistencia inmediata). Lo que involucra varios agregados se coordina aparte (eventos de dominio, consistencia eventual).
5. **Un repositorio por agregado**, no por tabla: `IPedidoRepository` guarda y carga el `Pedido` con sus líneas. No existe `ILineaPedidoRepository`.

### 2. Exponer la colección sin entregar el control

El patrón más común (y el de las notas de clase):

```csharp
private readonly List<LineaPedido> _lineas = new();
public IReadOnlyCollection<LineaPedido> Lineas => _lineas;   // ⚠️ casi
```

`IReadOnlyCollection<T>` no tiene `Add`, pero **sigue siendo la misma lista**. Un cast la abre:

```csharp
((List<LineaPedido>)pedido.Lineas).Add(lineaTrucha);   // compila y funciona
```

La forma segura es devolver un envoltorio de solo lectura:

```csharp
public IReadOnlyList<LineaPedido> Lineas => _lineas.AsReadOnly();   // ReadOnlyCollection<T>: el cast falla
```

`AsReadOnly()` crea una vista (no copia los elementos) cuyos métodos de escritura lanzan `NotSupportedException`. Además, el campo `_lineas` es `readonly`: nadie puede reemplazar la lista entera.

### 3. Los objetos internos se modifican solo a través de la raíz

`LineaPedido` es una entidad **local** al agregado: se identifica por su producto dentro del pedido, pero no tiene sentido fuera de él. Sus métodos de cambio son `internal` y la raíz decide cuándo llamarlos:

```csharp
public sealed class LineaPedido
{
    public Guid ProductoId { get; }
    public Dinero PrecioUnitario { get; }
    public int Cantidad { get; private set; }
    public Dinero Subtotal => PrecioUnitario * Cantidad;

    internal LineaPedido(Guid productoId, Dinero precio, int cantidad) { ... }

    internal void Sumar(int cantidad) { ... }       // solo el Pedido (mismo ensamblado) puede llamarlo
    internal void CambiarA(int cantidad) { ... }
}
```

`internal` es la herramienta de C# más cercana a "solo la raíz": el dominio suele vivir en su propio proyecto (ensamblado), así que la aplicación y la API no pueden llamar esos métodos.

### 4. El tamaño importa: agregados pequeños

El error más frecuente es meter todo en un agregado gigante: "el `Cliente` contiene sus `Pedidos`, que contienen sus `Pagos`...". Consecuencias:

* Para agregar una línea hay que cargar **todos** los pedidos del cliente.
* Dos usuarios que modifican pedidos distintos del mismo cliente **chocan** (conflictos de concurrencia).

La regla práctica: un agregado contiene solo lo necesario para proteger sus invariantes. `Pedido` y `Cliente` son agregados distintos, unidos por `ClienteId`.

### 5. ¿Y la lógica que no pertenece a ningún agregado?

Algunas reglas involucran varios agregados: "un cliente con deuda no puede confirmar pedidos". El `Pedido` no conoce la deuda del `Cliente`. Opciones:

* **Servicio de dominio:** una clase sin estado que coordina (`ServicioDeCredito.PuedeComprar(cliente, pedido)`).
* **Pasar el dato necesario a la raíz:** `pedido.Confirmar(tieneDeuda: cliente.TieneDeuda)`.
* **Eventos de dominio:** el pedido publica `PedidoConfirmado` y otro agregado reacciona después, en otra transacción.

-----

## Ejemplo completo

```csharp
var clienteId = Guid.NewGuid();
var tecladoId = Guid.NewGuid();
var mouseId = Guid.NewGuid();

var pedido = Pedido.Crear(clienteId, "PEN");
pedido.AgregarProducto(tecladoId, new Dinero(150m, "PEN"), 1);
pedido.AgregarProducto(mouseId, new Dinero(60m, "PEN"), 2);
pedido.AgregarProducto(tecladoId, new Dinero(150m, "PEN"), 1);   // mismo producto: se suma a la línea

foreach (var l in pedido.Lineas)
    Console.WriteLine($"  {l.ProductoId.ToString()[..8]} x{l.Cantidad} = {l.Subtotal}");
Console.WriteLine($"Líneas: {pedido.Lineas.Count} | Total: {pedido.Total}");

Intentar("Precio en otra moneda", () => pedido.AgregarProducto(Guid.NewGuid(), new Dinero(10m, "USD"), 1));
Intentar("Cantidad 0", () => pedido.CambiarCantidad(mouseId, 0));
Intentar("Abrir la lista con un cast", () => ((List<LineaPedido>)pedido.Lineas).Clear());
Intentar("Escribir en la vista", () => ((IList<LineaPedido>)pedido.Lineas).Clear());

pedido.QuitarProducto(mouseId);
pedido.Confirmar();
Console.WriteLine($"Confirmado. Total: {pedido.Total}");
Intentar("Agregar a un pedido confirmado", () => pedido.AgregarProducto(mouseId, new Dinero(60m, "PEN"), 1));

static void Intentar(string accion, Action operacion)
{
    try { operacion(); Console.WriteLine($"✔ {accion}"); }
    catch (Exception ex) when (ex is ArgumentException or DomainException or InvalidCastException or NotSupportedException)
    {
        Console.WriteLine($"✘ {accion}: {ex.GetType().Name} - {ex.Message.Split('\n')[0]}");
    }
}

class DomainException(string mensaje) : Exception(mensaje);

enum EstadoPedido { Borrador, Confirmado }

readonly record struct Dinero
{
    public decimal Monto { get; }
    public string Moneda { get; }

    public Dinero(decimal monto, string moneda)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(monto);
        ArgumentException.ThrowIfNullOrWhiteSpace(moneda);
        (Monto, Moneda) = (decimal.Round(monto, 2), moneda.Trim().ToUpperInvariant());
    }

    public static Dinero operator +(Dinero a, Dinero b)
    {
        if (a.Moneda != b.Moneda) throw new DomainException($"No se puede sumar {a.Moneda} con {b.Moneda}.");
        return new Dinero(a.Monto + b.Monto, a.Moneda);
    }

    public static Dinero operator *(Dinero a, int cantidad) => new(a.Monto * cantidad, a.Moneda);

    public override string ToString() => $"{Monto:N2} {Moneda}";
}

sealed class LineaPedido
{
    public const int MaximoPorLinea = 10;

    public Guid ProductoId { get; }
    public Dinero PrecioUnitario { get; }
    public int Cantidad { get; private set; }
    public Dinero Subtotal => PrecioUnitario * Cantidad;

    internal LineaPedido(Guid productoId, Dinero precioUnitario, int cantidad)
    {
        ProductoId = productoId;
        PrecioUnitario = precioUnitario;
        CambiarA(cantidad);
    }

    internal void Sumar(int cantidad) => CambiarA(Cantidad + cantidad);

    internal void CambiarA(int cantidad)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(cantidad);
        if (cantidad > MaximoPorLinea)
            throw new DomainException($"Máximo {MaximoPorLinea} unidades por producto.");
        Cantidad = cantidad;
    }
}

sealed class Pedido
{
    public const int MaximoLineas = 20;

    private readonly List<LineaPedido> _lineas = new();

    public Guid Id { get; }
    public Guid ClienteId { get; }            // referencia a otro agregado: solo el Id
    public string Moneda { get; }
    public EstadoPedido Estado { get; private set; } = EstadoPedido.Borrador;
    public IReadOnlyList<LineaPedido> Lineas => _lineas.AsReadOnly();
    public Dinero Total => _lineas.Aggregate(new Dinero(0, Moneda), (acc, l) => acc + l.Subtotal);

    private Pedido(Guid id, Guid clienteId, string moneda)
    {
        if (clienteId == Guid.Empty) throw new ArgumentException("El cliente es obligatorio.", nameof(clienteId));
        (Id, ClienteId, Moneda) = (id, clienteId, moneda.Trim().ToUpperInvariant());
    }

    public static Pedido Crear(Guid clienteId, string moneda) => new(Guid.CreateVersion7(), clienteId, moneda);

    public void AgregarProducto(Guid productoId, Dinero precio, int cantidad)
    {
        ExigirBorrador();
        if (precio.Moneda != Moneda)
            throw new DomainException($"El pedido es en {Moneda}; no acepta precios en {precio.Moneda}.");

        var existente = _lineas.Find(l => l.ProductoId == productoId);
        if (existente is not null)
        {
            existente.Sumar(cantidad);
            return;
        }

        if (_lineas.Count == MaximoLineas)
            throw new DomainException($"Un pedido no puede tener más de {MaximoLineas} líneas.");
        _lineas.Add(new LineaPedido(productoId, precio, cantidad));
    }

    public void CambiarCantidad(Guid productoId, int cantidad)
    {
        ExigirBorrador();
        BuscarLinea(productoId).CambiarA(cantidad);
    }

    public void QuitarProducto(Guid productoId)
    {
        ExigirBorrador();
        _lineas.Remove(BuscarLinea(productoId));
    }

    public void Confirmar()
    {
        ExigirBorrador();
        if (_lineas.Count == 0) throw new DomainException("No se puede confirmar un pedido vacío.");
        Estado = EstadoPedido.Confirmado;
    }

    private LineaPedido BuscarLinea(Guid productoId) =>
        _lineas.Find(l => l.ProductoId == productoId)
        ?? throw new DomainException("El producto no está en el pedido.");

    private void ExigirBorrador()
    {
        if (Estado != EstadoPedido.Borrador)
            throw new DomainException("El pedido ya no se puede modificar.");
    }
}
```

Salida (los Ids cambian en cada ejecución):

```text
  0b7f3c1e x2 = 300.00 PEN
  5d21a9f4 x2 = 120.00 PEN
Líneas: 2 | Total: 420.00 PEN
✘ Precio en otra moneda: DomainException - El pedido es en PEN; no acepta precios en USD.
✘ Cantidad 0: ArgumentOutOfRangeException - cantidad ('0') must be a non-negative and non-zero value. (Parameter 'cantidad')
✘ Abrir la lista con un cast: InvalidCastException - Unable to cast object of type 'System.Collections.ObjectModel.ReadOnlyCollection`1[LineaPedido]' to type 'System.Collections.Generic.List`1[LineaPedido]'.
✘ Escribir en la vista: NotSupportedException - Collection is read-only.
Confirmado. Total: 300.00 PEN
✘ Agregar a un pedido confirmado: DomainException - El pedido ya no se puede modificar.
```

Fíjate en lo que **no** se puede hacer desde afuera:

* Crear una `LineaPedido` (`new LineaPedido(...)` es `internal`; en un proyecto de dominio separado, no compila desde la API).
* Cambiar una cantidad sin pasar por el pedido.
* Modificar la lista, ni siquiera con un cast.
* Asignar el total: se calcula.

-----

## Errores comunes

**1. `IReadOnlyCollection<T> Items => _items;` y creer que está protegido.**
Qué pasa: `((List<T>)pedido.Items).Add(...)` funciona y salta todas las reglas.
Por qué: la interfaz oculta métodos, pero el objeto sigue siendo la `List<T>` original.
Arreglo: `_items.AsReadOnly()` (o devolver una copia, si la colección es chica).

**2. Modificar objetos internos sin pasar por la raíz.**
Qué pasa: `pedido.Lineas[0].Cantidad = 500` (si `Cantidad` tuviera `set` público) o un repositorio de líneas.
Por qué: los objetos internos tienen setters públicos o su propio repositorio.
Arreglo: setters privados, métodos de cambio `internal`, un repositorio por agregado.

**3. Referenciar otro agregado por objeto.**
Qué pasa: `pedido.Cliente.Bloquear()` modifica el agregado `Cliente` dentro de la transacción del pedido; además, cargar un pedido arrastra el cliente entero.
Por qué: se modelan relaciones como en la base de datos (navegaciones por todos lados).
Arreglo: `public Guid ClienteId { get; }`; si hace falta el cliente, se carga desde su propio repositorio.

**4. Agregados gigantes.**
Qué pasa: cargas lentas y conflictos de concurrencia constantes.
Por qué: se agrupó por "relación" y no por "invariantes que deben protegerse juntas".
Arreglo: agregados pequeños; preguntar qué regla **exige** que estos objetos cambien en la misma transacción.

**5. Total almacenado y actualizado a mano (`Total += precio`).**
Qué pasa: al quitar o cambiar una línea, el total queda desincronizado; tampoco hay validación de moneda ni de cantidad.
Por qué: el valor derivado se mantiene manualmente en varios métodos.
Arreglo: calcularlo a partir de las líneas, o recalcularlo en **un** método privado llamado por todas las operaciones.

**6. Constructor público que deja el agregado incompleto.**
Qué pasa: `new Pedido()` sin cliente ni moneda.
Por qué: el constructor no exige los datos mínimos.
Arreglo: constructor privado y un método de fábrica (`Pedido.Crear(clienteId, moneda)`) que exija lo obligatorio.

-----

## Según la versión de C#

* **C# 1:** `internal`, la base para limitar el acceso a los objetos internos.
* **.NET 2.0:** `List<T>.AsReadOnly()` y `ReadOnlyCollection<T>`.
* **.NET 4.5:** `IReadOnlyCollection<T>` e `IReadOnlyList<T>`.
* **C# 12:** constructores primarios (`class DomainException(string mensaje) : Exception(mensaje)`) y expresiones de colección (`[]`).
* **.NET 9:** `Guid.CreateVersion7()`.

-----

## Cuándo sí y cuándo no

**Modela un agregado cuando:**

* Hay reglas que involucran a varios objetos a la vez (líneas de un pedido, asientos de una reserva, cuotas de un préstamo).
* Esos objetos no tienen sentido fuera del todo (una línea sin pedido).

**No lo fuerces cuando:**

* Cada entidad tiene reglas solo propias: cada una puede ser su propio agregado de una sola entidad (lo más común).
* La relación es solo de "pertenencia" sin reglas compartidas (un cliente y sus pedidos): agregados separados, unidos por Id.

-----

## Resumen en 5 líneas

1. Un agregado es un grupo de objetos que cambian juntos, con una raíz como única puerta de entrada.
2. La raíz protege las invariantes de todo el grupo; los objetos internos solo se modifican a través de ella.
3. Expón colecciones con `AsReadOnly()`, no con un simple `IReadOnlyCollection<T> => _lista`.
4. Entre agregados, referencia por Id; una transacción modifica un solo agregado.
5. Agregados pequeños y un repositorio por agregado.

-----

## Para profundizar

<details>
<summary>Repositorios por agregado</summary>

```csharp
public interface IPedidoRepository
{
    Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct = default);   // carga el pedido CON sus líneas
    Task AgregarAsync(Pedido pedido, CancellationToken ct = default);
    Task GuardarCambiosAsync(CancellationToken ct = default);               // una transacción, un agregado
}
```

La interfaz vive en el dominio (o en la capa de aplicación); la implementación con EF Core vive en infraestructura. No hay `ILineaPedidoRepository`: las líneas se cargan y guardan siempre con su pedido.

</details>

<details>
<summary>Eventos de dominio y consistencia eventual</summary>

Si al confirmar un pedido hay que reservar stock en el agregado `Producto`, no se modifican los dos en la misma transacción. El pedido registra un evento (`PedidoConfirmado`) en una lista interna; después de guardar, un manejador lo procesa y actualiza el stock en otra transacción. El sistema queda consistente **eventualmente** (milisegundos o segundos después), y cada agregado se mantiene pequeño e independiente. Si la reserva falla, se compensa (por ejemplo, cancelando el pedido).

</details>

<details>
<summary>Concurrencia optimista</summary>

Como el agregado se guarda entero, dos usuarios que lo cargan a la vez pueden pisarse. La solución habitual es un número de versión (`RowVersion` / token de concurrencia en EF Core): al guardar, si la versión cambió desde que se leyó, se lanza `DbUpdateConcurrencyException` y la operación se reintenta o se informa al usuario.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un agregado es un grupo de objetos que se tratan como una unidad, como un pedido con sus líneas. La entidad principal, la raíz, es la única que se usa desde afuera: para agregar una línea llamas a `pedido.AgregarLinea(...)` y el pedido valida sus reglas. La lista de líneas se expone de solo lectura para que nadie la modifique directamente.

### Respuesta ampliada (semi-senior)

El agregado define una frontera de consistencia: todo lo que está adentro se modifica a través de la raíz, que protege invariantes que involucran a varios objetos, y se persiste en una sola transacción mediante un repositorio por agregado. Entre agregados se referencia por identidad y se coordina con eventos de dominio y consistencia eventual. Diseño agregados pequeños, guiados por las invariantes y no por las relaciones de la base de datos, para evitar cargas pesadas y conflictos de concurrencia, que manejo con concurrencia optimista. En C# expongo las colecciones con `AsReadOnly()` (un `IReadOnlyCollection` sobre la lista se puede castear), uso campos `readonly`, constructores privados con factories y métodos `internal` en las entidades internas.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué solo se accede por la raíz?**
Porque las invariantes involucran a varios objetos: si se modifica uno por separado, nadie verifica que el conjunto siga siendo válido.

**2. ¿Por qué referenciar otros agregados por Id?**
Para no modificar dos agregados en la misma transacción, no cargar grafos enormes y mantener las fronteras claras.

**3. ¿Cómo decides el tamaño de un agregado?**
Por las invariantes que deben cumplirse de forma inmediata. Lo que puede ser consistente eventualmente va en otro agregado.

**4. ¿`IReadOnlyCollection<T>` hace inmutable la colección?**
No. Solo oculta los métodos; con un cast se recupera la lista original. `AsReadOnly()` sí bloquea la escritura.

-----

## Práctica

**Ejercicio 1.** Modela el agregado `Reserva` de un cine: tiene `FuncionId` (referencia por Id), una lista de asientos (`Asiento`, value object con `Fila` y `Numero`) y un estado. Reglas: máximo 6 asientos, sin repetidos, no se modifica una reserva pagada. Expón los asientos de forma segura.

<details>
<summary>Solución</summary>

```csharp
var reserva = Reserva.Crear(Guid.NewGuid());
reserva.AgregarAsiento(new Asiento('F', 7));
reserva.AgregarAsiento(new Asiento('F', 8));
Console.WriteLine(string.Join(", ", reserva.Asientos));   // F7, F8

try { reserva.AgregarAsiento(new Asiento('F', 7)); }
catch (InvalidOperationException ex) { Console.WriteLine(ex.Message); }

record Asiento(char Fila, int Numero)
{
    public override string ToString() => $"{Fila}{Numero}";
}

enum EstadoReserva { Pendiente, Pagada }

sealed class Reserva
{
    private const int MaximoAsientos = 6;
    private readonly List<Asiento> _asientos = [];

    public Guid Id { get; }
    public Guid FuncionId { get; }
    public EstadoReserva Estado { get; private set; }
    public IReadOnlyList<Asiento> Asientos => _asientos.AsReadOnly();

    private Reserva(Guid id, Guid funcionId) => (Id, FuncionId) = (id, funcionId);

    public static Reserva Crear(Guid funcionId) => new(Guid.CreateVersion7(), funcionId);

    public void AgregarAsiento(Asiento asiento)
    {
        if (Estado == EstadoReserva.Pagada) throw new InvalidOperationException("La reserva ya está pagada.");
        if (_asientos.Contains(asiento)) throw new InvalidOperationException($"El asiento {asiento} ya está en la reserva.");
        if (_asientos.Count == MaximoAsientos) throw new InvalidOperationException($"Máximo {MaximoAsientos} asientos.");
        _asientos.Add(asiento);
    }

    public void Pagar()
    {
        if (_asientos.Count == 0) throw new InvalidOperationException("No se puede pagar una reserva vacía.");
        Estado = EstadoReserva.Pagada;
    }
}
```

`_asientos.Contains(asiento)` funciona porque `Asiento` es un record: dos `F7` son iguales por valor. (Que el asiento no esté reservado por **otra** reserva es una regla entre agregados: se resuelve en el agregado `Funcion` o en un servicio, no aquí.)

</details>

**Ejercicio 2.** ¿Cliente y Pedido deberían ser un solo agregado (el cliente con su lista de pedidos)? Justifica.

<details>
<summary>Solución</summary>

No. No hay una invariante que exija modificar el cliente y sus pedidos en la misma transacción. Juntarlos obligaría a cargar todos los pedidos para agregar uno, y dos compras simultáneas del mismo cliente generarían conflictos de concurrencia. Son dos agregados; el pedido guarda `ClienteId`. Reglas como "un cliente con deuda no compra" se resuelven pasando el dato al pedido o con un servicio de dominio.

</details>

-----

## Siguiente lección

Terminaste los bloques tácticos básicos. Para practicarlos con los ejercicios de clase: [Ejercicios de DDD](Ejercicios.md). Vuelve al [índice del módulo](README.md).
