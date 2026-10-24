# Puertos y adaptadores

## En una frase

En la arquitectura hexagonal, la aplicación se comunica con el mundo solo a través de **puertos** (interfaces que ella define): por los puertos de **entrada** la usan (HTTP, consola, colas, pruebas) y por los de **salida** usa lo que necesita (base, pagos, email); los **adaptadores** traducen entre cada tecnología y esos puertos, y se pueden cambiar sin tocar el núcleo.

-----

## Antes de empezar

Conviene que ya sepas:

* La regla de dependencias y la raíz de composición: [Capas y la regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md).
* Commands, queries y handlers: [CQRS y mediator con MediatR](../06-cqrs-y-mediator/02-CQRS%20y%20mediator%20con%20MediatR.md).
* Dobles de prueba: [Dobles de prueba con Moq](../../05-testing-y-calidad/01-pruebas-unitarias/03-Dobles%20de%20prueba%20con%20Moq.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Puerto:** interfaz definida por la aplicación para comunicarse con el exterior.
* **Puerto de entrada (*driving* / primario):** cómo el exterior le pide cosas a la aplicación (los casos de uso).
* **Puerto de salida (*driven* / secundario):** lo que la aplicación necesita del exterior (persistencia, pagos, mensajería).
* **Adaptador:** implementación que conecta una tecnología concreta con un puerto.
* **Actor primario y secundario:** quien inicia una interacción (un usuario, un job) frente a quien la aplicación usa (una base, un proveedor de pagos).

-----

## El problema

El caso de uso "confirmar y cobrar un pedido" vive dentro de un controlador:

```csharp
[HttpPost("{codigo}/confirmar")]
public async Task<IActionResult> Confirmar(string codigo)
{
    var pedido = await _db.Pedidos.Include(p => p.Lineas).SingleOrDefaultAsync(p => p.Codigo == codigo);
    if (pedido is null) return NotFound();

    var respuesta = await _http.PostAsJsonAsync("https://api.pasarela.com/cobros", new { pedido.Total });
    if (!respuesta.IsSuccessStatusCode) return BadRequest("Pago rechazado");

    pedido.Confirmar();
    await _db.SaveChangesAsync();
    return NoContent();
}
```

Meses después, el negocio pide:

1. **Un job nocturno** que confirme en lote los pedidos pagados por transferencia. La lógica está en un controlador: se copia y pega en el job, con sus diferencias.
2. **Una importación desde una cola** de un sistema de terceros. Tercera copia.
3. **Cambiar de proveedor de pagos.** La URL y el formato están dentro del controlador (y de sus copias). Hay que tocar los tres lugares.
4. **Probar el flujo** "pago rechazado → no se confirma" exige levantar ASP.NET Core y simular un `HttpClient`.

Las consecuencias:

* un caso de uso atado a un mecanismo de entrada (HTTP);
* lógica duplicada por cada nueva forma de usar la aplicación;
* proveedores externos incrustados en la lógica.

-----

## Cómo funciona

### 1. El hexágono

```text
                    ADAPTADORES DE ENTRADA                     ADAPTADORES DE SALIDA
                    (driving / primarios)                      (driven / secundarios)

  Endpoint HTTP ──┐                                        ┌──► Repositorio EF Core ──► PostgreSQL
  Job nocturno ───┤      ┌───────────────────────────┐     │
  Consumidor cola ┼────► │ Puerto de entrada          │     ├──► Cliente pasarela ──► API de pagos
  Prueba xUnit ───┘      │  IConfirmarPedido          │     │
                         │                            │ ────┤
                         │   APLICACIÓN + DOMINIO     │     └──► Enviador SMTP ──► servidor de email
                         │                            │
                         │ Puertos de salida          │
                         │  IPedidoRepository         │
                         │  IPasarelaPago             │
                         └───────────────────────────┘
```

El hexágono no tiene nada de especial con el número seis: es una forma de dibujar que **todos los lados son iguales**. HTTP no es más importante que un job ni que una prueba: todos entran por el mismo puerto.

### 2. Puertos de entrada: los casos de uso

```csharp
public interface IConfirmarPedido
{
    Task<ResultadoConfirmacion> EjecutarAsync(string codigo, CancellationToken ct);
}
```

* Definen **qué** ofrece la aplicación, con sus propios tipos (`ResultadoConfirmacion`, no `IActionResult`).
* Muchas veces no hace falta la interfaz: la clase del caso de uso (o el command de MediatR) **es** el puerto. La interfaz aporta cuando quieres reemplazar el caso de uso entero en pruebas de un adaptador de entrada.

### 3. Puertos de salida: lo que la aplicación necesita

```csharp
public interface IPasarelaPago
{
    Task<bool> CobrarAsync(string referencia, decimal monto, CancellationToken ct);
}
```

* Hablan el idioma de la aplicación: "cobrar un monto", no "hacer POST a /v2/charges con un JSON".
* Son pocos y estrechos: la aplicación pide lo mínimo que necesita (*interface segregation*).

### 4. Adaptadores: la traducción

| Adaptador | Puerto | Traduce |
| --- | --- | --- |
| Endpoint `POST /pedidos/{codigo}/confirmar` | `IConfirmarPedido` (entrada) | HTTP → llamada al caso de uso; resultado → `204`/`404`/`422` |
| Job nocturno | `IConfirmarPedido` (entrada) | Lista de códigos → llamadas; resultados → log |
| `PedidoRepositoryEf` | `IPedidoRepository` (salida) | Agregado ↔ tablas con EF Core |
| `PasarelaStripe` | `IPasarelaPago` (salida) | `CobrarAsync` → API HTTP del proveedor y sus errores |

Los adaptadores son "tontos" respecto del negocio: traducen formatos y protocolos, no deciden reglas. Si un adaptador empieza a tener `if` sobre reglas de negocio, esa lógica se escapó del núcleo.

### 5. Adaptadores de prueba

Una prueba es un adaptador de entrada más, y un doble es un adaptador de salida más:

```csharp
var confirmar = new ConfirmarPedido(new RepositorioEnMemoria(), new PasarelaQueRechaza());
var resultado = await confirmar.EjecutarAsync("P-001", ct);
Assert.False(resultado.Exito);
```

Probar "pago rechazado → no se confirma" no necesita HTTP, base ni proveedor: el núcleo se prueba por sus puertos. Es la mayor ventaja práctica de la arquitectura hexagonal.

### 6. Hexagonal y Clean: dos dibujos de la misma idea

| Hexagonal | Clean |
| --- | --- |
| Núcleo de la aplicación | Dominio + casos de uso |
| Puertos | Interfaces de la capa de aplicación |
| Adaptadores de entrada | Controladores / presentación |
| Adaptadores de salida | Infraestructura (repositorios, gateways) |

Clean agrega la distinción entre dominio y casos de uso dentro del núcleo; hexagonal pone el foco en la simetría entre entradas y salidas. En .NET suelen combinarse: proyectos de Clean, con vocabulario de puertos y adaptadores.

-----

## Ejemplo completo

Consola con un caso de uso usado por **dos** adaptadores de entrada y con **dos** adaptadores de salida. `Program.cs`:

```csharp
using System.Globalization;
using Tienda.Adaptadores.Entrada;
using Tienda.Adaptadores.Salida;
using Tienda.Aplicacion;
using Tienda.Dominio;

// Raíz de composición
var repositorio = new RepositorioEnMemoria();
var pasarela = new PasarelaFalsa(montoMaximo: 500m);
IConfirmarPedido confirmar = new ConfirmarPedido(repositorio, pasarela);

repositorio.Agregar(Pedido.Crear("P-001", ("Teclado", 150m, 1), ("Mouse", 60m, 2)));   // 270
repositorio.Agregar(Pedido.Crear("P-002", ("Monitor", 600m, 1)));                     // 600: la pasarela lo rechaza
repositorio.Agregar(Pedido.Crear("P-003"));                                           // vacío

// Adaptador de entrada 1: comandos de consola
var consola = new AdaptadorConsola(confirmar);
await consola.ProcesarAsync("confirmar P-001");
await consola.ProcesarAsync("confirmar P-999");

// Adaptador de entrada 2: job por lotes
var lote = new JobConfirmacionLote(confirmar);
await lote.EjecutarAsync(["P-002", "P-003"]);

Console.WriteLine($"Cobros realizados: {string.Join(", ", pasarela.Cobrados)}");

// ---------------- Dominio ----------------
namespace Tienda.Dominio
{
    public sealed class DomainException(string mensaje) : Exception(mensaje);

    public sealed record LineaPedido(string Producto, decimal PrecioUnitario, int Cantidad);

    public sealed class Pedido
    {
        private readonly List<LineaPedido> _lineas = [];

        private Pedido(string codigo) => Codigo = codigo;

        public string Codigo { get; }
        public bool Confirmado { get; private set; }
        public decimal Total => _lineas.Sum(l => l.PrecioUnitario * l.Cantidad);

        public static Pedido Crear(string codigo, params (string Producto, decimal Precio, int Cantidad)[] lineas)
        {
            var pedido = new Pedido(codigo);
            foreach (var (producto, precio, cantidad) in lineas)
            {
                pedido._lineas.Add(new LineaPedido(producto, precio, cantidad));
            }
            return pedido;
        }

        public void AsegurarQuePuedeConfirmarse()
        {
            if (Confirmado) throw new DomainException("El pedido ya está confirmado.");
            if (_lineas.Count == 0) throw new DomainException("No se puede confirmar un pedido vacío.");
        }

        public void Confirmar()
        {
            AsegurarQuePuedeConfirmarse();
            Confirmado = true;
        }
    }
}

// ---------------- Aplicación: puertos y caso de uso ----------------
namespace Tienda.Aplicacion
{
    using Tienda.Dominio;

    // Puerto de entrada
    public interface IConfirmarPedido
    {
        Task<ResultadoConfirmacion> EjecutarAsync(string codigo, CancellationToken ct = default);
    }

    // Puertos de salida
    public interface IPedidoRepository
    {
        Task<Pedido?> ObtenerAsync(string codigo, CancellationToken ct);
        Task GuardarAsync(Pedido pedido, CancellationToken ct);
    }

    public interface IPasarelaPago
    {
        Task<bool> CobrarAsync(string referencia, decimal monto, CancellationToken ct);
    }

    public sealed record ResultadoConfirmacion(bool Exito, string Detalle);

    public sealed class ConfirmarPedido(IPedidoRepository repositorio, IPasarelaPago pasarela) : IConfirmarPedido
    {
        public async Task<ResultadoConfirmacion> EjecutarAsync(string codigo, CancellationToken ct = default)
        {
            Pedido? pedido = await repositorio.ObtenerAsync(codigo, ct);
            if (pedido is null)
            {
                return new(false, "no existe");
            }

            try
            {
                pedido.AsegurarQuePuedeConfirmarse();   // reglas del dominio antes de cobrar
            }
            catch (DomainException ex)
            {
                return new(false, $"error de dominio: {ex.Message}");
            }

            string monto = pedido.Total.ToString("F2", CultureInfo.InvariantCulture);
            if (!await pasarela.CobrarAsync(pedido.Codigo, pedido.Total, ct))
            {
                return new(false, $"rechazado por la pasarela ({monto})");
            }

            pedido.Confirmar();
            await repositorio.GuardarAsync(pedido, ct);
            return new(true, $"confirmado y cobrado ({monto})");
        }
    }
}

// ---------------- Adaptadores de entrada: usan el puerto de entrada ----------------
namespace Tienda.Adaptadores.Entrada
{
    using Tienda.Aplicacion;

    public sealed class AdaptadorConsola(IConfirmarPedido confirmar)
    {
        public async Task ProcesarAsync(string linea)
        {
            string[] partes = linea.Split(' ', StringSplitOptions.RemoveEmptyEntries);
            if (partes is not ["confirmar", var codigo])
            {
                Console.WriteLine($"[consola] comando no reconocido: {linea}");
                return;
            }

            ResultadoConfirmacion r = await confirmar.EjecutarAsync(codigo);
            Console.WriteLine($"[consola] {codigo}: {r.Detalle}");
        }
    }

    public sealed class JobConfirmacionLote(IConfirmarPedido confirmar)
    {
        public async Task EjecutarAsync(IEnumerable<string> codigos)
        {
            int confirmados = 0, total = 0;
            foreach (string codigo in codigos)
            {
                total++;
                ResultadoConfirmacion r = await confirmar.EjecutarAsync(codigo);
                if (r.Exito) confirmados++;
                Console.WriteLine($"[lote] {codigo}: {r.Detalle}");
            }
            Console.WriteLine($"[lote] resumen: {confirmados} de {total} confirmados");
        }
    }
}

// ---------------- Adaptadores de salida: implementan los puertos de salida ----------------
namespace Tienda.Adaptadores.Salida
{
    using Tienda.Aplicacion;
    using Tienda.Dominio;

    public sealed class RepositorioEnMemoria : IPedidoRepository
    {
        private readonly Dictionary<string, Pedido> _pedidos = [];

        public void Agregar(Pedido pedido) => _pedidos[pedido.Codigo] = pedido;

        public Task<Pedido?> ObtenerAsync(string codigo, CancellationToken ct) =>
            Task.FromResult(_pedidos.GetValueOrDefault(codigo));

        public Task GuardarAsync(Pedido pedido, CancellationToken ct)
        {
            _pedidos[pedido.Codigo] = pedido;
            return Task.CompletedTask;
        }
    }

    // Simula un proveedor que rechaza montos altos
    public sealed class PasarelaFalsa(decimal montoMaximo) : IPasarelaPago
    {
        public List<string> Cobrados { get; } = [];

        public Task<bool> CobrarAsync(string referencia, decimal monto, CancellationToken ct)
        {
            if (monto > montoMaximo)
            {
                return Task.FromResult(false);
            }
            Cobrados.Add(referencia);
            return Task.FromResult(true);
        }
    }
}
```

Salida:

```text
[consola] P-001: confirmado y cobrado (270.00)
[consola] P-999: no existe
[lote] P-002: rechazado por la pasarela (600.00)
[lote] P-003: error de dominio: No se puede confirmar un pedido vacío.
[lote] resumen: 0 de 2 confirmados
Cobros realizados: P-001
```

Qué observar:

* La consola y el job son **dos adaptadores de entrada** del mismo puerto: la lógica de confirmar y cobrar existe una sola vez.
* El caso de uso no sabe si lo llamó un humano, un job o una prueba, ni si la pasarela es real o falsa.
* El pedido vacío falla **antes** de cobrar: el caso de uso consulta al dominio primero, así no se cobra algo que no se puede confirmar.
* Cambiar de proveedor de pagos es escribir otro adaptador de `IPasarelaPago` y cambiar una línea en la raíz de composición.
* Mañana, un endpoint de ASP.NET Core sería un tercer adaptador de entrada de unas pocas líneas.

-----

## Errores comunes

**1. Puertos que filtran la tecnología.**
Qué pasa: `IPasarelaPago.CobrarAsync(StripeChargeRequest)` o un puerto que devuelve `HttpResponseMessage`; cambiar de proveedor obliga a cambiar la aplicación.
Por qué: el puerto se diseñó desde el adaptador, no desde la necesidad.
Arreglo: tipos propios de la aplicación; el adaptador traduce a y desde el proveedor.

**2. Lógica de negocio en los adaptadores.**
Qué pasa: el endpoint decide "si el total supera 1000, pedir aprobación"; el job no lo hace y el comportamiento difiere.
Por qué: el adaptador hace algo más que traducir.
Arreglo: la regla va al dominio o al caso de uso; el adaptador solo convierte formatos y protocolos.

**3. Puertos enormes.**
Qué pasa: un `IInfraestructura` con veinte métodos que todos implementan a medias.
Por qué: se agrupó por tecnología en lugar de por necesidad.
Arreglo: puertos chicos por capacidad (`IPasarelaPago`, `IEnviadorEmail`, `IReloj`).

**4. Adaptadores de salida que llaman a adaptadores de entrada (o al revés).**
Qué pasa: el repositorio invoca un caso de uso, o el endpoint usa el `DbContext` directamente "para una consulta rápida".
Por qué: se saltea el núcleo.
Arreglo: todo pasa por los puertos. Si una consulta de lectura necesita ir directo a la base por rendimiento, hazlo como una decisión explícita (un *query handler* con Dapper es un adaptador más), no como un atajo.

**5. Una interfaz de entrada para cada caso de uso "por las dudas".**
Qué pasa: `IConfirmarPedido`, `ICrearPedido`, `IObtenerPedido`... con una sola implementación cada una y ningún beneficio.
Por qué: se aplica el patrón al pie de la letra.
Arreglo: la clase del caso de uso (o el command de MediatR) ya es el puerto de entrada; agrega la interfaz cuando necesites reemplazarla.

-----

## Según la versión de .NET

* **2005:** Alistair Cockburn publica *Hexagonal Architecture*, con el nombre alternativo *Ports and Adapters*.
* **2008–2012:** Onion y Clean adoptan la misma idea con capas concéntricas.
* **ASP.NET Core:** los endpoints (controladores o Minimal APIs) son adaptadores de entrada naturales; `BackgroundService` y los consumidores de colas, otros.
* **.NET 8:** `TimeProvider` en el framework: el reloj, un puerto de salida clásico, ya viene abstraído.

-----

## Cuándo sí y cuándo no

**Piensa en puertos y adaptadores cuando:** la aplicación tiene (o tendrá) varias formas de entrada (API, jobs, colas), depende de proveedores externos que pueden cambiar, o quieres probar el núcleo sin infraestructura.

**No los multipliques cuando:** hay un solo mecanismo de entrada, un solo proveedor sin perspectivas de cambio y poca lógica: un puerto por cada dependencia agrega indirección sin beneficio.

**Mantén siempre al menos esto:** los proveedores externos (pagos, email, APIs de terceros) detrás de un puerto propio. Es barato y cambia la vida el día que hay que reemplazarlos o probarlos.

-----

## Resumen en 5 líneas

1. La aplicación habla con el exterior solo por puertos que ella define.
2. Puertos de entrada: cómo la usan (casos de uso). Puertos de salida: lo que necesita (base, pagos, email).
3. Los adaptadores traducen tecnología ↔ puertos y no deciden reglas.
4. HTTP, jobs, colas y pruebas son adaptadores de entrada equivalentes.
5. Cambiar un proveedor es escribir un adaptador nuevo y cambiar la composición.

-----

## Para profundizar

<details>
<summary>Capa anticorrupción en el adaptador de salida</summary>

Cuando el sistema externo tiene un modelo muy distinto o confuso (un ERP con códigos crípticos), el adaptador funciona como **capa anticorrupción** (término de DDD): traduce su modelo al tuyo y evita que sus conceptos se filtren al dominio. Si el proveedor devuelve `status: "C3"`, el adaptador lo convierte en `ResultadoCobro.RechazadoPorFondos`, y el núcleo nunca ve "C3".

</details>

<details>
<summary>Los mismos puertos, distintos adaptadores por entorno</summary>

En desarrollo puedes usar `EnviadorEmailConsola`; en pruebas de integración, uno que guarda en memoria; en producción, el SMTP real. La raíz de composición elige según el entorno (`if (builder.Environment.IsDevelopment())`). Es la misma idea que reemplazar servicios con `ConfigureTestServices` en las [pruebas de integración](../../05-testing-y-calidad/02-pruebas-de-integracion/02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

En la arquitectura hexagonal, la aplicación define puertos (interfaces) y la tecnología se conecta mediante adaptadores. Hay puertos de entrada, como los casos de uso que llama un controlador, y de salida, como el repositorio o la pasarela de pagos, que implementa la infraestructura.

### Respuesta ampliada (semi-senior)

El núcleo expone puertos de entrada (casos de uso) y define puertos de salida con su propio lenguaje, estrechos y por capacidad. Endpoints, jobs, consumidores de colas y pruebas son adaptadores de entrada equivalentes; repositorios, clientes de proveedores y enviadores de email son adaptadores de salida que traducen sin decidir reglas, a veces como capa anticorrupción. Gracias a eso pruebo el núcleo con adaptadores falsos y cambio proveedores escribiendo un adaptador nuevo. Es la misma idea que Clean, con foco en la simetría entre entradas y salidas, y no creo interfaces de entrada para cada caso de uso salvo que las necesite.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es un puerto de salida?**
Una interfaz que la aplicación define para algo que necesita del exterior, como persistir o cobrar.

**2. ¿Un controlador es un adaptador?**
Sí, de entrada: traduce HTTP a llamadas a casos de uso y sus resultados a códigos de estado.

**3. ¿Por qué un job y un endpoint deberían usar el mismo puerto?**
Para que la lógica exista una sola vez y se comporte igual sin importar quién la invoque.

**4. ¿Dónde va la lógica de negocio?**
En el núcleo (dominio y casos de uso), nunca en los adaptadores.

-----

## Práctica

**Ejercicio 1.** Escribe una prueba xUnit del caso de uso para "pago rechazado → el pedido no queda confirmado" usando adaptadores falsos, sin Moq.

<details>
<summary>Solución</summary>

```csharp
[Fact]
public async Task Confirmar_PagoRechazado_NoConfirmaNiGuarda()
{
    var repositorio = new RepositorioEnMemoria();
    var pedido = Pedido.Crear("P-010", ("Monitor", 600m, 1));
    repositorio.Agregar(pedido);
    var confirmar = new ConfirmarPedido(repositorio, new PasarelaFalsa(montoMaximo: 500m));

    ResultadoConfirmacion resultado = await confirmar.EjecutarAsync("P-010");

    Assert.False(resultado.Exito);
    Assert.False(pedido.Confirmado);
}
```

La prueba es un adaptador de entrada y `PasarelaFalsa` un adaptador de salida. No hay HTTP, base ni biblioteca de mocks: el núcleo se prueba por sus puertos.

</details>

**Ejercicio 2.** Agrega un tercer adaptador de entrada: un endpoint de Minimal API `POST /pedidos/{codigo}/confirmar` que devuelva `204`, `404` o `422`. ¿Qué cambia en la aplicación?

<details>
<summary>Solución</summary>

```csharp
app.MapPost("/pedidos/{codigo}/confirmar", async (string codigo, IConfirmarPedido confirmar, CancellationToken ct) =>
{
    ResultadoConfirmacion r = await confirmar.EjecutarAsync(codigo, ct);
    if (r.Exito) return Results.NoContent();
    return r.Detalle == "no existe"
        ? Results.NotFound()
        : Results.Problem(detail: r.Detalle, statusCode: StatusCodes.Status422UnprocessableEntity);
});
```

La aplicación **no cambia**. Solo aparece un adaptador nuevo y su registro en DI. Ojo con un detalle: decidir el código HTTP comparando el texto `"no existe"` es frágil. Es mejor que `ResultadoConfirmacion` tenga un tipo de resultado explícito (`enum MotivoFallo { NoExiste, ReglaDeDominio, PagoRechazado }`) que el adaptador traduzca a `404`/`422`/`402`.

</details>

-----

## Siguiente lección

[Estructura de una solución Clean en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md)
