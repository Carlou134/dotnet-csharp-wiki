# Capas y la regla de dependencias

## En una frase

Clean Architecture se reduce a una regla: **las dependencias del código apuntan hacia adentro, hacia el dominio**; las reglas del negocio no conocen la base de datos, el framework web ni ninguna biblioteca de infraestructura, y son estas las que dependen de interfaces definidas por el núcleo.

-----

## Antes de empezar

Conviene que ya sepas:

* Entidades, value objects y agregados: [Domain-Driven Design táctico](../01-domain-driven-design/README.md).
* SOLID, en especial el principio de inversión de dependencias: [Código limpio](../../01-csharp-core-and-runtime/11-codigo-limpio/README.md).
* Interfaces e inyección por constructor: [Inyección de dependencias y ciclos de vida](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Arquitectura en capas (N-capas):** organización en presentación, negocio y datos, donde cada capa depende de la de abajo.
* **Regla de dependencias:** el código fuente solo puede depender de capas más internas; nunca de las externas.
* **Inversión de dependencias (DIP):** los módulos de alto nivel no dependen de los de bajo nivel; ambos dependen de abstracciones, y la abstracción pertenece al de alto nivel.
* **Dominio:** el modelo del negocio: entidades, value objects, reglas e invariantes.
* **Caso de uso:** una operación de la aplicación desde el punto de vista del usuario ("confirmar pedido"), que orquesta el dominio.
* **Infraestructura:** detalles técnicos: base de datos, archivos, HTTP, colas, email.
* **Raíz de composición (*composition root*):** el único lugar donde se conectan las implementaciones concretas con sus interfaces.

-----

## El problema

Una tienda organizada en las tres capas "de toda la vida":

```text
Tienda.Web  ──referencia──►  Tienda.Negocio  ──referencia──►  Tienda.Datos (EF Core, SQL Server)
```

```csharp
// Tienda.Datos: la entidad es una clase de EF Core
[Table("Pedidos")]
public class Pedido
{
    public int Id { get; set; }
    public int Estado { get; set; }
    public virtual List<LineaPedido> Lineas { get; set; } = new();
}

// Tienda.Negocio: depende de Tienda.Datos
public class PedidoService(TiendaDbContext db)
{
    public void Confirmar(int id)
    {
        var pedido = db.Pedidos.Include(p => p.Lineas).Single(p => p.Id == id);
        if (pedido.Lineas.Count == 0) throw new Exception("Pedido vacío");
        pedido.Estado = 2;
        db.SaveChanges();
    }
}
```

Parece ordenado. En la práctica:

1. **Las reglas del negocio dependen de EF Core.** Para probar "no se confirma un pedido vacío" hace falta un `DbContext`, o sea, una base de datos (o un mock de EF Core, que miente).
2. **La entidad la diseña la base.** `Estado` es un `int` porque así está la columna; `Lineas` es una lista pública con `set` porque EF Core la llena. Cualquier capa puede hacer `pedido.Estado = 2` sin pasar por ninguna regla.
3. **Cambiar un detalle técnico toca el negocio.** Migrar de SQL Server a PostgreSQL, o leer pedidos de una API externa, obliga a modificar `Tienda.Negocio` porque está atado a `TiendaDbContext`.
4. **La lógica se escapa hacia arriba.** Como el "negocio" es solo pasar datos, la validación termina en los controladores, duplicada.

Las consecuencias:

* reglas de negocio imposibles de probar sin infraestructura;
* un modelo del dominio condicionado por el esquema de la base;
* cambios técnicos que obligan a tocar y volver a probar las reglas.

-----

## Cómo funciona

### 1. Invertir la flecha

```text
N-capas clásico                          Clean / Hexagonal
Web ──► Negocio ──► Datos                Web ──► Aplicación ──► Dominio
                                                    ▲
                                         Infraestructura ──┘  (implementa interfaces de Aplicación)
```

En N-capas, el negocio depende de los datos. En Clean, **la infraestructura depende del núcleo**: la aplicación define la interfaz que necesita (`IPedidoRepository`) y la infraestructura la implementa con EF Core. El flujo de ejecución sigue yendo "hacia la base"; lo que cambia es la dirección de las **referencias en el código**.

### 2. Los círculos

```text
┌──────────────────────────────────────────────────────────┐
│  Frameworks y drivers: ASP.NET Core, EF Core, SMTP, colas  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Adaptadores: endpoints, repositorios, mapeos          │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  Aplicación: casos de uso, puertos (interfaces)   │ │ │
│  │  │  ┌──────────────────────────────────────────────┐ │ │ │
│  │  │  │  Dominio: entidades, value objects, reglas    │ │ │ │
│  │  │  └──────────────────────────────────────────────┘ │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
         Las dependencias del código solo apuntan hacia adentro
```

| Capa | Contiene | Puede depender de | No puede depender de |
| --- | --- | --- | --- |
| Dominio | `Pedido`, `Dinero`, `DomainException` | Nada (solo .NET base) | Aplicación, EF Core, ASP.NET Core |
| Aplicación | `ConfirmarPedido`, `IPedidoRepository`, DTOs | Dominio | Infraestructura, ASP.NET Core |
| Infraestructura | `PedidoRepositoryEf`, `TiendaDbContext`, cliente SMTP | Aplicación, Dominio | Web |
| Presentación (Web) | Endpoints, controladores | Aplicación (y la composición conoce todo) | — |

### 3. La interfaz pertenece a quien la usa

```csharp
// Aplicación: define lo que NECESITA, con sus términos
public interface IPedidoRepository
{
    Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct);
    Task GuardarAsync(Pedido pedido, CancellationToken ct);
}

// Infraestructura: cumple el contrato con la tecnología que sea
public sealed class PedidoRepositoryEf(TiendaDbContext db) : IPedidoRepository { ... }
```

Esto es la **inversión de dependencias**: la abstracción la escribe el módulo de alto nivel (la aplicación) y el de bajo nivel (EF Core) se adapta. Si la interfaz expusiera `IQueryable<Pedido>` o `DbSet`, la abstracción sería de EF Core disfrazada y la regla se rompería igual.

### 4. El dominio no sabe que existe una base

```csharp
public sealed class Pedido
{
    private readonly List<LineaPedido> _lineas = [];
    public Guid Id { get; }
    public EstadoPedido Estado { get; private set; }
    public IReadOnlyCollection<LineaPedido> Lineas => _lineas.AsReadOnly();

    public void Confirmar()
    {
        if (_lineas.Count == 0) throw new DomainException("No se puede confirmar un pedido vacío.");
        Estado = EstadoPedido.Confirmado;
    }
}
```

Sin atributos de EF Core, sin `virtual`, sin `set` públicos. El mapeo a tablas se configura **afuera**, en infraestructura, con Fluent API ([lección 3](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md)). Probar "no se confirma un pedido vacío" es crear un `Pedido` con `new` y llamar a `Confirmar()`.

### 5. La raíz de composición

Alguien tiene que decir "`IPedidoRepository` es `PedidoRepositoryEf`". Ese lugar es el **más externo**: `Program.cs` de la API. Es el único que conoce todas las capas, y lo hace solo para conectarlas:

```csharp
builder.Services.AddScoped<IPedidoRepository, PedidoRepositoryEf>();
builder.Services.AddScoped<ConfirmarPedido>();
```

### 6. Lo que Clean no es

* **No es "cuatro proyectos".** Es una dirección de dependencias. Puede vivir en un solo proyecto con carpetas, si alguien la respeta (y la [lección 4](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md) muestra cómo verificarlo).
* **No es "una interfaz por clase".** Las interfaces van donde hay una frontera con algo externo o variable, no delante de cada servicio.
* **No es independencia total de todo.** El dominio depende de C# y de la biblioteca base de .NET; no tiene sentido abstraer `List<T>` o `DateOnly`.
* **No lo inventó un solo autor.** Hexagonal (Cockburn, 2005), Onion (Palermo, 2008) y Clean (Martin, 2012) dicen esencialmente lo mismo: el núcleo en el centro y los detalles afuera.

-----

## Ejemplo completo

Consola con las capas como espacios de nombres en un solo archivo, para ver la regla sin la ceremonia de varios proyectos. `Program.cs`:

```csharp
using System.Globalization;
using Tienda.Aplicacion;
using Tienda.Dominio;
using Tienda.Infraestructura;

// Raíz de composición: el único lugar que conoce las implementaciones concretas
IPedidoRepository repositorio = new PedidoRepositoryEnMemoria();
var confirmar = new ConfirmarPedido(repositorio);

var conLineas = Pedido.Crear(clienteId: 7);
conLineas.AgregarLinea("Teclado", 150m, 1);
conLineas.AgregarLinea("Mouse", 60m, 2);
await repositorio.GuardarAsync(conLineas, CancellationToken.None);

var vacio = Pedido.Crear(clienteId: 8);
await repositorio.GuardarAsync(vacio, CancellationToken.None);

foreach (var id in new[] { conLineas.Id, vacio.Id, Guid.Empty })
{
    ResultadoConfirmacion resultado = await confirmar.EjecutarAsync(id, CancellationToken.None);
    Console.WriteLine(resultado.Mensaje);
}

// ---------------- Dominio: no referencia a nadie ----------------
namespace Tienda.Dominio
{
    public enum EstadoPedido { Borrador, Confirmado }

    public sealed class DomainException(string mensaje) : Exception(mensaje);

    public sealed record LineaPedido(string Producto, decimal PrecioUnitario, int Cantidad)
    {
        public decimal Subtotal => PrecioUnitario * Cantidad;
    }

    public sealed class Pedido
    {
        private readonly List<LineaPedido> _lineas = [];

        private Pedido(Guid id, int clienteId) => (Id, ClienteId) = (id, clienteId);

        public Guid Id { get; }
        public int ClienteId { get; }
        public EstadoPedido Estado { get; private set; } = EstadoPedido.Borrador;
        public IReadOnlyCollection<LineaPedido> Lineas => _lineas.AsReadOnly();
        public decimal Total => _lineas.Sum(l => l.Subtotal);

        public static Pedido Crear(int clienteId) => new(Guid.NewGuid(), clienteId);

        public void AgregarLinea(string producto, decimal precio, int cantidad)
        {
            if (Estado != EstadoPedido.Borrador) throw new DomainException("El pedido ya no admite cambios.");
            ArgumentOutOfRangeException.ThrowIfNegativeOrZero(cantidad);
            _lineas.Add(new LineaPedido(producto, precio, cantidad));
        }

        public void Confirmar()
        {
            if (_lineas.Count == 0) throw new DomainException("No se puede confirmar un pedido vacío.");
            Estado = EstadoPedido.Confirmado;
        }
    }
}

// ---------------- Aplicación: depende solo del Dominio ----------------
namespace Tienda.Aplicacion
{
    using Tienda.Dominio;

    // Puerto: la aplicación define lo que necesita
    public interface IPedidoRepository
    {
        Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct);
        Task GuardarAsync(Pedido pedido, CancellationToken ct);
    }

    public sealed record ResultadoConfirmacion(bool Exito, string Mensaje);

    // Caso de uso: orquesta; las reglas están en el dominio
    public sealed class ConfirmarPedido(IPedidoRepository repositorio)
    {
        public async Task<ResultadoConfirmacion> EjecutarAsync(Guid pedidoId, CancellationToken ct)
        {
            Pedido? pedido = await repositorio.ObtenerAsync(pedidoId, ct);
            if (pedido is null)
            {
                return new(false, $"No existe el pedido.");
            }

            try
            {
                pedido.Confirmar();
            }
            catch (DomainException ex)
            {
                return new(false, $"Error de dominio: {ex.Message}");
            }

            await repositorio.GuardarAsync(pedido, ct);
            return new(true, $"Pedido confirmado. Total: {pedido.Total.ToString("F2", CultureInfo.InvariantCulture)}");
        }
    }
}

// ---------------- Infraestructura: implementa el puerto de Aplicación ----------------
namespace Tienda.Infraestructura
{
    using Tienda.Aplicacion;
    using Tienda.Dominio;

    // Hoy en memoria; mañana EF Core. Aplicación y Dominio no cambian.
    public sealed class PedidoRepositoryEnMemoria : IPedidoRepository
    {
        private readonly Dictionary<Guid, Pedido> _pedidos = [];

        public Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct) =>
            Task.FromResult(_pedidos.GetValueOrDefault(id));

        public Task GuardarAsync(Pedido pedido, CancellationToken ct)
        {
            _pedidos[pedido.Id] = pedido;
            return Task.CompletedTask;
        }
    }
}
```

Salida:

```text
Pedido confirmado. Total: 270.00
Error de dominio: No se puede confirmar un pedido vacío.
No existe el pedido.
```

Qué observar:

* `Tienda.Dominio` no tiene ningún `using` de las otras capas: no sabe que existen.
* `Tienda.Aplicacion` usa `Tienda.Dominio` y **define** `IPedidoRepository`. No conoce ninguna implementación.
* `Tienda.Infraestructura` depende de la aplicación (implementa su interfaz). Es la flecha invertida.
* Solo las primeras líneas (la raíz de composición) conocen `PedidoRepositoryEnMemoria`. Reemplazarlo por uno con EF Core es cambiar **una línea** ahí.
* La regla "pedido vacío" vive en `Pedido.Confirmar()`; el caso de uso solo orquesta (cargar, ejecutar, guardar) y traduce el resultado.
* En un solo proyecto, nada **impide** que alguien agregue `using Tienda.Infraestructura;` dentro del dominio. Por eso, en proyectos reales, las capas suelen ser proyectos separados: el compilador hace cumplir la regla.

-----

## Errores comunes

**1. Interfaces que exponen la tecnología.**
Qué pasa: `IPedidoRepository` devuelve `IQueryable<Pedido>` o recibe `Expression<Func<...>>`, y la aplicación termina escribiendo consultas de EF Core.
Por qué: la abstracción copia la forma de la herramienta en lugar de expresar la necesidad del negocio.
Arreglo: métodos con intención (`ObtenerAsync(id)`, `PendientesDeEnvioAsync()`), definidos por la aplicación.

**2. Entidades del dominio con atributos de persistencia.**
Qué pasa: `[Table]`, `[Column]`, `virtual` y `set` públicos en el dominio; el modelo lo dicta la base.
Por qué: se reutiliza la entidad de EF Core como entidad de dominio sin separar el mapeo.
Arreglo: dominio limpio y mapeo con Fluent API (`IEntityTypeConfiguration`) en infraestructura.

**3. Reglas de negocio en los casos de uso.**
Qué pasa: `ConfirmarPedido` comprueba "si no tiene líneas, error" y `Pedido` es una bolsa de propiedades: el modelo anémico, ahora con más carpetas.
Por qué: se confunde "orquestar" con "decidir".
Arreglo: el caso de uso carga, llama al dominio y guarda; las reglas e invariantes viven en las entidades.

**4. La aplicación referencia ASP.NET Core.**
Qué pasa: casos de uso que devuelven `IActionResult` o leen `HttpContext`; no se pueden usar desde un worker ni probar sin servidor.
Por qué: la capa de aplicación quedó atada a un mecanismo de entrega.
Arreglo: devolver datos o resultados de la aplicación; la presentación los traduce a HTTP.

**5. Creer que las carpetas garantizan la regla.**
Qué pasa: con el tiempo, el dominio termina con un `using` de infraestructura "solo para una cosa".
Por qué: nada lo impide en un único proyecto.
Arreglo: proyectos separados (el compilador lo impide) o pruebas de arquitectura ([lección 4](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md)).

-----

## Según la versión de .NET

* **2005:** Alistair Cockburn publica *Hexagonal Architecture (Ports and Adapters)*.
* **2008:** Jeffrey Palermo describe *Onion Architecture*, con ejemplos en .NET.
* **2012:** Robert C. Martin publica *The Clean Architecture*, que unifica las anteriores; el libro llega en 2017.
* **ASP.NET Core 1.0 (2016):** inyección de dependencias integrada, lo que hace natural la raíz de composición en `Program.cs`.
* **EF Core 2.x–8:** campos de respaldo, constructores privados, conversiones de valor y *complex types*: el dominio puede ser limpio y aun así mapearse sin atributos.
* **C# 12:** constructores primarios; casos de uso con dependencias en una línea.

-----

## Cuándo sí y cuándo no

**Aplica la regla de dependencias cuando:** el sistema tiene reglas de negocio reales que deben probarse sin infraestructura, va a vivir años, o es probable que cambien detalles técnicos (base, proveedores externos, mecanismos de entrega).

**No la apliques con todo su andamiaje cuando:** es un CRUD sin reglas, un prototipo o un servicio chico que transforma datos: las capas solo agregarían archivos que pasan datos de un lado a otro.

**Recuerda que la regla escala hacia abajo:** incluso en un proyecto chico, que el dominio no conozca EF Core cuesta poco y rinde mucho.

-----

## Resumen en 5 líneas

1. Las dependencias del código apuntan hacia adentro: Web → Aplicación → Dominio, e Infraestructura → Aplicación.
2. La aplicación define las interfaces que necesita; la infraestructura las implementa.
3. El dominio no conoce bases, frameworks ni atributos de persistencia.
4. Los casos de uso orquestan; las reglas viven en el dominio.
5. La raíz de composición, en la capa más externa, es el único lugar que conecta implementaciones.

-----

## Para profundizar

<details>
<summary>¿La interfaz del repositorio va en Dominio o en Aplicación?</summary>

En el módulo de [Agregados](../01-domain-driven-design/04-Agregados.md) se mencionaba que puede ir en cualquiera de las dos. Con DDD, se suele poner en el dominio, porque el repositorio es "la colección de agregados" del modelo. Con hexagonal, en la aplicación, porque es un puerto que el caso de uso necesita. Las dos cumplen la regla de dependencias (la implementación sigue en infraestructura). Elige una y sé consistente en todo el proyecto.

</details>

<details>
<summary>El flujo de control frente a la dependencia del código</summary>

En tiempo de ejecución, la petición viaja Web → Aplicación → Infraestructura → base, y la respuesta vuelve. En el código fuente, Infraestructura apunta a Aplicación. Esa diferencia la resuelve el polimorfismo: la aplicación llama a `IPedidoRepository` y, en ejecución, recibe un `PedidoRepositoryEf`. Es exactamente lo que hace la inyección de dependencias: decidir en tiempo de ejecución qué implementación concreta cumple cada interfaz.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Clean Architecture separa el código en capas (dominio, aplicación, infraestructura y presentación) y la regla principal es que las dependencias apuntan hacia el dominio. El dominio no conoce la base de datos: la aplicación define interfaces como el repositorio y la infraestructura las implementa.

### Respuesta ampliada (semi-senior)

La esencia es la regla de dependencias: el núcleo (dominio y casos de uso) no referencia detalles técnicos; la infraestructura implementa interfaces que define la aplicación, lo que es inversión de dependencias. Así pruebo las reglas sin base de datos, mapeo el dominio con Fluent API sin contaminarlo y puedo cambiar detalles sin tocar el negocio. Los casos de uso orquestan y el dominio decide. Lo conecto todo en la raíz de composición. No lo confundo con "cuatro proyectos y una interfaz por clase": la uso cuando hay reglas que proteger, y en sistemas simples prefiero algo más liviano.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay con la arquitectura en tres capas?**
En tres capas el negocio depende de los datos; en Clean, los datos dependen del núcleo a través de interfaces.

**2. ¿Dónde se registran las implementaciones?**
En la raíz de composición, en la capa más externa (normalmente `Program.cs`).

**3. ¿El dominio puede usar EF Core?**
No. El mapeo se configura en infraestructura con Fluent API.

**4. ¿Clean, Onion y Hexagonal son lo mismo?**
Variantes de la misma idea: núcleo en el centro, detalles afuera, dependencias hacia adentro.

-----

## Práctica

**Ejercicio 1.** Clasifica cada tipo en su capa y di cuáles violan la regla: (a) `Pedido` con `[Table("Pedidos")]`; (b) `IEnviadorEmail` usado por `ConfirmarPedido`; (c) `EnviadorEmailSmtp`; (d) `ConfirmarPedido` que devuelve `IResult`; (e) `Dinero` (value object).

<details>
<summary>Solución</summary>

* (a) Dominio, **pero viola** la regla: `[Table]` es de persistencia. Va sin atributos y se mapea en infraestructura.
* (b) Aplicación: es un puerto que el caso de uso necesita.
* (c) Infraestructura: implementa `IEnviadorEmail` con SMTP.
* (d) Aplicación, **pero viola** la regla: `IResult` es de ASP.NET Core. Debe devolver un resultado propio y que la presentación lo traduzca.
* (e) Dominio.

</details>

**Ejercicio 2.** Agrega al ejemplo un `IEnviadorEmail` que `ConfirmarPedido` use para avisar al cliente, con una implementación en consola en infraestructura. ¿Qué capas tocas y cuáles no?

<details>
<summary>Solución</summary>

En `Tienda.Aplicacion`:

```csharp
public interface IEnviadorEmail
{
    Task EnviarAsync(int clienteId, string asunto, CancellationToken ct);
}

public sealed class ConfirmarPedido(IPedidoRepository repositorio, IEnviadorEmail email)
{
    // ... después de guardar:
    // await email.EnviarAsync(pedido.ClienteId, "Pedido confirmado", ct);
}
```

En `Tienda.Infraestructura`:

```csharp
public sealed class EnviadorEmailConsola : IEnviadorEmail
{
    public Task EnviarAsync(int clienteId, string asunto, CancellationToken ct)
    {
        Console.WriteLine($"[email a cliente {clienteId}] {asunto}");
        return Task.CompletedTask;
    }
}
```

En la raíz de composición: `new ConfirmarPedido(repositorio, new EnviadorEmailConsola())`.

Tocas Aplicación (puerto y caso de uso), Infraestructura (adaptador) y la composición. **El dominio no cambia**: enviar un email no es una regla del pedido.

</details>

-----

## Siguiente lección

[Puertos y adaptadores](02-Puertos%20y%20adaptadores.md)
