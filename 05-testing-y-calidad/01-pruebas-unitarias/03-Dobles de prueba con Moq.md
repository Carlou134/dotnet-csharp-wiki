# Dobles de prueba con Moq

## En una frase

Un doble de prueba reemplaza una dependencia lenta, externa o no determinista para que la prueba controle lo que esa dependencia devuelve y observe cómo se la usa; Moq los crea a partir de interfaces, pero mockear de más produce pruebas que se rompen con cada refactor sin encontrar errores.

-----

## Antes de empezar

Conviene que ya sepas:

* Pruebas con xUnit y `Assert.ThrowsAsync`: [xUnit: Fact, Theory y aserciones](02-xUnit%20Fact%20Theory%20y%20aserciones.md).
* Interfaces e inyección por constructor: [Inyección de dependencias y ciclos de vida](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Doble de prueba (*test double*):** cualquier objeto que reemplaza a una dependencia real durante una prueba.
* **Stub:** doble que devuelve respuestas preparadas; la prueba verifica el resultado del SUT.
* **Mock:** doble sobre el que la prueba verifica **interacciones** (qué métodos se llamaron y con qué argumentos).
* **Fake:** implementación funcional simplificada (un repositorio en memoria, SQLite en memoria).
* **Moq:** biblioteca de .NET que crea stubs y mocks a partir de interfaces en tiempo de ejecución.
* **Verificación de interacción:** comprobar que el SUT llamó (o no llamó) a una dependencia de cierta forma.

-----

## El problema

Una guía de clase muestra dos pruebas con Moq:

```csharp
// 1. Lector de archivos
var mockFileReader = new Mock<IFileReaderConnector>();
mockFileReader.Setup(m => m.readString(It.IsAny<string>())).Returns("readingFile");

var result = stringOperation.ReadFile("archivo.txt", mockFileReader.Object);
Assert.Equal("readingFile", result);

// 2. Logger
var mockLogger = new Mock<ILogger>();
var ops = new StringOperations(mockLogger.Object);
Assert.Equal(3, ops.CountAllCurrencies("hello Plexi", 'l'));
```

Y en otro proyecto, el equipo mockea todo lo que encuentra: el `DbContext`, el `HttpClient`, las entidades, hasta una calculadora de impuestos sin dependencias.

Lo que pasa:

1. **La prueba 1 no prueba nada.** `ReadFile` solo devuelve lo que le da el lector. La prueba configura el mock para devolver `"readingFile"` y verifica que el resultado sea `"readingFile"`: está probando a Moq. Si `ReadFile` tuviera lógica (recortar, validar la ruta), esta prueba no la ejerce.
2. **La prueba 2 está bien en lo que hace**, pero un `Mock<ILogger>` no sirve para verificar los logs: `LogInformation` y compañía son métodos de **extensión**, y Moq no puede interceptarlos. Para "un logger que no moleste" existe `NullLogger`.
3. **Mockear el `DbContext`** obliga a simular `DbSet`, `IQueryable` y `SaveChangesAsync`. Las pruebas pasan y en producción falla una consulta que EF Core no sabe traducir: el mock nunca tradujo nada.
4. **Cada refactor rompe veinte pruebas**: se verificó cada llamada interna (`Verify(... Times.Once)` por todos lados), así que cambiar el *cómo* rompe pruebas aunque el *qué* siga correcto.

Las consecuencias:

* pruebas que pasan sin ejercer la lógica;
* confianza falsa sobre código de infraestructura;
* una suite que frena los refactors en lugar de habilitarlos.

-----

## Cómo funciona

### 1. Tipos de dobles

| Doble | Qué hace | Qué verificas | Ejemplo |
| --- | --- | --- | --- |
| Dummy | Llena un parámetro que no se usa | Nada | `NullLogger<T>.Instance` |
| Stub | Devuelve respuestas preparadas | El **resultado** del SUT | Pasarela que siempre aprueba |
| Mock | Registra llamadas | Las **interacciones** | "Se guardó el pedido una vez" |
| Fake | Implementación real simplificada | El resultado | Repositorio en memoria, SQLite en memoria |

En la práctica, "mock" se usa para todo; la distinción importa al decidir **qué verificar**.

### 2. Moq en cuatro operaciones

```csharp
var pasarela = new Mock<IPasarelaPago>();

// Stub: qué devolver
pasarela.Setup(p => p.CobrarAsync(7, 120m, It.IsAny<CancellationToken>())).ReturnsAsync(true);

// Stub de error
pasarela.Setup(p => p.CobrarAsync(It.IsAny<int>(), It.IsAny<decimal>(), It.IsAny<CancellationToken>()))
        .ThrowsAsync(new HttpRequestException("Timeout"));

// Usar el objeto falso
var servicio = new ServicioPedidos(repo.Object, pasarela.Object, NullLogger<ServicioPedidos>.Instance);

// Mock: verificar la interacción
repo.Verify(r => r.GuardarAsync(It.IsAny<Pedido>(), It.IsAny<CancellationToken>()), Times.Never);
```

| Pieza | Para |
| --- | --- |
| `It.IsAny<T>()` | Cualquier valor de ese parámetro |
| `It.Is<T>(x => condición)` | Valores que cumplen una condición |
| `Returns` / `ReturnsAsync` | Valor de retorno (síncrono / `Task`) |
| `Throws` / `ThrowsAsync` | Simular una falla |
| `Callback(...)` | Capturar argumentos o ejecutar algo al llamarse |
| `Verify(..., Times.Once)` | Comprobar cuántas veces se llamó |

Un mock "suelto" (el comportamiento por defecto) devuelve valores por defecto para lo que no configuraste: `null`, `0`, `false`, o un `Task` completado con el valor por defecto en métodos asíncronos.

### 3. Qué reemplazar y qué no

```text
¿La dependencia es...
  lenta, externa o no determinista (red, pagos, email, reloj, disco)?  → doble (stub/mock)
  tu propia lógica de dominio (entidades, value objects, calculadoras)? → la REAL
  infraestructura que no es tuya (DbContext, HttpClient)?               → no la mockees:
        envuélvela detrás de una interfaz propia, o prueba con integración (SQLite, Testcontainers)
```

* **No mockees lo que no es tuyo.** Un mock de `DbContext` reproduce lo que crees que hace EF Core, no lo que hace.
* **No mockees value objects ni entidades.** Créalos con `new`: son baratos y deterministas.
* **El reloj sí:** `TimeProvider` (con `FakeTimeProvider` del paquete `Microsoft.Extensions.TimeProvider.Testing`) hace repetibles las pruebas con fechas.

### 4. Verificar resultados antes que interacciones

| Verificas | La prueba se rompe cuando... | Ejemplo |
| --- | --- | --- |
| El resultado / estado | Cambia el **comportamiento** | `Assert.Equal(EstadoPedido.Pagado, pedido.Estado)` |
| Una interacción | Cambia la **implementación** | `Verify(r => r.GuardarAsync(...), Times.Once)` |

Verifica interacciones solo cuando **son** el comportamiento: "se cobró al cliente", "se envió el email", "no se guardó nada si el pago falló". No verifiques cada llamada intermedia.

### 5. `ILogger` en pruebas

| Necesitas | Usa |
| --- | --- |
| Que el logger no moleste | `NullLogger<T>.Instance` (`Microsoft.Extensions.Logging.Abstractions`) |
| Verificar qué se registró | `FakeLogger<T>` (`Microsoft.Extensions.Diagnostics.Testing`, .NET 8+) |
| `Mock<ILogger<T>>` con `Verify` | Evítalo: hay que verificar el método genérico `Log<TState>` con `It.IsAnyType`; es frágil e ilegible |

### 6. Una dependencia pasada como parámetro no se prueba sola

La prueba del lector de archivos de la guía verifica que "el método devuelve lo que devuelve el lector". Una prueba útil ejerce **lógica propia** alrededor de la dependencia:

```csharp
// Si ReadFile recorta espacios y lanza si el archivo está vacío, eso sí vale la pena probar:
lector.Setup(l => l.LeerTexto("config.txt")).Returns("  valor  \n");
Assert.Equal("valor", ops.LeerConfiguracion("config.txt", lector.Object));
```

Si el método no tiene lógica propia, no necesita una prueba unitaria.

### 7. Sobre Moq y sus alternativas

En 2023, Moq 4.20.0 incluyó SponsorLink, un componente que leía datos de configuración de Git del desarrollador; tras las críticas, se quitó en la versión 4.20.2. Desde entonces, muchos equipos evalúan **NSubstitute** (`sustituto.CobrarAsync(...).Returns(true)`) o **FakeItEasy**. Los conceptos de esta lección son los mismos en las tres.

-----

## Ejemplo completo

Misma solución que en las lecciones anteriores. El proyecto `src/Tienda` necesita el paquete `Microsoft.Extensions.Logging.Abstractions` 10.0.0, y el de pruebas, `Moq` 4.20.72.

`src/Tienda/ServicioPedidos.cs`:

```csharp
using Microsoft.Extensions.Logging;

namespace Tienda;

public enum EstadoPedido { Pendiente, Pagado }

public sealed class Pedido
{
    public int Id { get; init; }
    public decimal Total { get; init; }
    public EstadoPedido Estado { get; private set; } = EstadoPedido.Pendiente;

    public void MarcarPagado()
    {
        if (Estado != EstadoPedido.Pendiente)
        {
            throw new InvalidOperationException($"El pedido {Id} ya está {Estado}.");
        }
        Estado = EstadoPedido.Pagado;
    }
}

public interface IRepositorioPedidos
{
    Task<Pedido?> ObtenerAsync(int id, CancellationToken ct);
    Task GuardarAsync(Pedido pedido, CancellationToken ct);
}

public interface IPasarelaPago
{
    Task<bool> CobrarAsync(int pedidoId, decimal monto, CancellationToken ct);
}

public sealed class ServicioPedidos(IRepositorioPedidos repositorio, IPasarelaPago pasarela, ILogger<ServicioPedidos> logger)
{
    public async Task<bool> PagarAsync(int pedidoId, CancellationToken ct = default)
    {
        Pedido pedido = await repositorio.ObtenerAsync(pedidoId, ct)
            ?? throw new InvalidOperationException($"El pedido {pedidoId} no existe.");

        bool aprobado = await pasarela.CobrarAsync(pedido.Id, pedido.Total, ct);
        if (!aprobado)
        {
            logger.LogWarning("Pago rechazado para el pedido {PedidoId}", pedido.Id);
            return false;
        }

        pedido.MarcarPagado();
        await repositorio.GuardarAsync(pedido, ct);
        return true;
    }
}
```

`tests/Tienda.Tests/ServicioPedidosTests.cs`:

```csharp
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Tienda;

namespace Tienda.Tests;

public class ServicioPedidosTests
{
    private readonly Mock<IRepositorioPedidos> _repositorio = new();
    private readonly Mock<IPasarelaPago> _pasarela = new();
    private readonly Pedido _pedido = new() { Id = 7, Total = 120m };   // entidad REAL, no un mock
    private readonly ServicioPedidos _servicio;

    public ServicioPedidosTests()
    {
        _repositorio.Setup(r => r.ObtenerAsync(7, It.IsAny<CancellationToken>())).ReturnsAsync(_pedido);
        _servicio = new ServicioPedidos(_repositorio.Object, _pasarela.Object, NullLogger<ServicioPedidos>.Instance);
    }

    [Fact]
    public async Task PagarAsync_PagoAprobado_MarcaPagadoYGuarda()
    {
        _pasarela.Setup(p => p.CobrarAsync(7, 120m, It.IsAny<CancellationToken>())).ReturnsAsync(true);

        bool resultado = await _servicio.PagarAsync(7);

        Assert.True(resultado);
        Assert.Equal(EstadoPedido.Pagado, _pedido.Estado);
        _repositorio.Verify(r => r.GuardarAsync(_pedido, It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task PagarAsync_PagoRechazado_NoCambiaElEstadoNiGuarda()
    {
        _pasarela.Setup(p => p.CobrarAsync(7, 120m, It.IsAny<CancellationToken>())).ReturnsAsync(false);

        bool resultado = await _servicio.PagarAsync(7);

        Assert.False(resultado);
        Assert.Equal(EstadoPedido.Pendiente, _pedido.Estado);
        _repositorio.Verify(r => r.GuardarAsync(It.IsAny<Pedido>(), It.IsAny<CancellationToken>()), Times.Never);
    }

    [Fact]
    public async Task PagarAsync_PasarelaCaida_PropagaLaExcepcionYNoGuarda()
    {
        _pasarela.Setup(p => p.CobrarAsync(It.IsAny<int>(), It.IsAny<decimal>(), It.IsAny<CancellationToken>()))
                 .ThrowsAsync(new HttpRequestException("Timeout"));

        await Assert.ThrowsAsync<HttpRequestException>(() => _servicio.PagarAsync(7));

        _repositorio.Verify(r => r.GuardarAsync(It.IsAny<Pedido>(), It.IsAny<CancellationToken>()), Times.Never);
    }

    [Fact]
    public async Task PagarAsync_PedidoInexistente_LanzaYNoCobra()
    {
        await Assert.ThrowsAsync<InvalidOperationException>(() => _servicio.PagarAsync(99));

        _pasarela.Verify(p => p.CobrarAsync(It.IsAny<int>(), It.IsAny<decimal>(), It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

Ejecución con `dotnet test`. Resultado: **4 pruebas superadas, 0 con error** (el formato del resumen depende de la versión del SDK y del idioma).

Qué observar:

* `IPasarelaPago` es un **stub** en tres pruebas (respuesta preparada) y un **mock** en la última (se verifica que **no** se llamó).
* `Pedido` es la entidad real: su regla `MarcarPagado` se ejerce de verdad, y el estado se verifica con `Assert`, no con `Verify`.
* Los `Verify` que hay son el comportamiento importante: "si el pago falla, no se guarda" y "si el pedido no existe, no se cobra".
* `ObtenerAsync(99)` no está configurado: el mock suelto devuelve un `Task` completado con `null`, y el servicio lanza `InvalidOperationException`.
* `NullLogger<ServicioPedidos>.Instance` cumple el parámetro sin agregar nada a la prueba.

-----

## Errores comunes

**1. Probar que el mock devuelve lo que configuraste.**
Qué pasa: pruebas verdes que no ejercen ninguna lógica propia.
Por qué: el método bajo prueba solo reenvía el valor de la dependencia.
Arreglo: prueba la lógica alrededor de la dependencia; si no hay, no hace falta prueba unitaria.

**2. Mockear `DbContext`, `DbSet` o `HttpClient`.**
Qué pasa: pruebas complicadas que pasan mientras producción falla (consultas no traducibles, configuración HTTP).
Por qué: el mock simula tu idea del comportamiento, no el comportamiento real.
Arreglo: interfaz propia sobre la infraestructura y pruebas de integración para esa implementación (SQLite en memoria, Testcontainers, `HttpMessageHandler` falso).

**3. Verificar cada llamada interna.**
Qué pasa: cualquier refactor rompe muchas pruebas sin que haya un error.
Por qué: las pruebas quedaron acopladas a la implementación.
Arreglo: verifica resultados y estado; `Verify` solo para interacciones que son el comportamiento.

**4. `Mock<ILogger>` para verificar logs.**
Qué pasa: `NotSupportedException: Unsupported expression ... Extension methods ... may not be used in setup / verification expressions`.
Por qué: `LogInformation` es un método de extensión; Moq solo intercepta miembros de la interfaz.
Arreglo: `NullLogger<T>` si no importan los logs; `FakeLogger<T>` si quieres verificarlos.

**5. Mockear clases concretas no virtuales.**
Qué pasa: `NotSupportedException: Unsupported expression ... Non-overridable members ... may not be used in setup / verification expressions`.
Por qué: Moq genera una subclase en tiempo de ejecución y solo puede sobrescribir miembros virtuales o de interfaz.
Arreglo: depende de una interfaz o usa la clase real.

-----

## Según la versión de .NET

* **Moq 4 (2010):** API con expresiones lambda (`Setup`, `Verify`, `It.IsAny`).
* **Moq 4.20.0 (2023):** incluyó SponsorLink; **4.20.2** lo quitó.
* **.NET 8:** `TimeProvider` en el framework, `FakeTimeProvider` y `FakeLogger` en paquetes de pruebas oficiales.
* **NSubstitute 5 / FakeItEasy 8:** alternativas activas con la misma idea y otra sintaxis.

-----

## Cuándo sí y cuándo no

**Usa dobles cuando:** la dependencia es externa, lenta o no determinista (pagos, email, colas, reloj, servicios HTTP), o quieres simular fallas difíciles de provocar (timeouts, rechazos).

**Usa fakes cuando:** el comportamiento de la dependencia importa (un repositorio en memoria que de verdad guarda y devuelve) y quieres pruebas menos acopladas que con `Verify`.

**No uses dobles cuando:** la dependencia es tu propia lógica de dominio, un value object o infraestructura que no es tuya: usa la real o una prueba de integración.

-----

## Resumen en 5 líneas

1. Stub devuelve respuestas; mock verifica interacciones; fake es una implementación simplificada.
2. Moq: `Setup`/`Returns`/`Throws` para preparar, `Verify` para comprobar llamadas.
3. Reemplaza lo externo y no determinista; usa reales para tu dominio; no mockees lo que no es tuyo.
4. Verifica resultados antes que interacciones; `Verify` solo cuando la llamada es el comportamiento.
5. `NullLogger` o `FakeLogger` para logs; nunca `Mock<ILogger>` con `Verify`.

-----

## Para profundizar

<details>
<summary>Capturar argumentos con <code>Callback</code></summary>

```csharp
Pedido? guardado = null;
_repositorio.Setup(r => r.GuardarAsync(It.IsAny<Pedido>(), It.IsAny<CancellationToken>()))
            .Callback<Pedido, CancellationToken>((p, _) => guardado = p)
            .Returns(Task.CompletedTask);

await _servicio.PagarAsync(7);

Assert.NotNull(guardado);
Assert.Equal(EstadoPedido.Pagado, guardado.Estado);
```

Útil cuando el SUT construye el objeto que pasa a la dependencia y quieres verificar su contenido con asserts normales, en lugar de un `It.Is<T>` con una condición larga.

</details>

<details>
<summary>Mocks estrictos</summary>

`new Mock<IPasarelaPago>(MockBehavior.Strict)` lanza una excepción ante cualquier llamada no configurada. Detecta llamadas inesperadas, pero obliga a configurar todo, y las pruebas se vuelven aún más acopladas a la implementación. El comportamiento suelto (`Loose`, por defecto) suele ser mejor punto de partida.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un mock es un objeto falso que reemplaza una dependencia, como una base de datos o un servicio externo, para que la prueba sea rápida y aislada. Con Moq configuro qué devuelve con `Setup` y `Returns`, y verifico llamadas con `Verify`.

### Respuesta ampliada (semi-senior)

Distingo stubs (respuestas preparadas), mocks (verificación de interacciones) y fakes (implementaciones simplificadas). Reemplazo solo lo externo o no determinista; uso las clases reales de mi dominio y no mockeo lo que no es mío, como `DbContext` o `HttpClient`: lo envuelvo en una interfaz propia y pruebo la implementación con integración. Verifico primero resultados y estado, y uso `Verify` solo cuando la interacción es el comportamiento, para no acoplar las pruebas a la implementación. Para logs uso `NullLogger` o `FakeLogger`, y para fechas, `FakeTimeProvider`. Conozco el episodio de SponsorLink de Moq y alternativas como NSubstitute.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre stub y mock?**
El stub da respuestas y verificas el resultado del SUT; en el mock verificas cómo se lo llamó.

**2. ¿Por qué no mockear el `DbContext`?**
Porque el mock no traduce consultas ni aplica restricciones: las pruebas pasan aunque EF Core falle en producción.

**3. ¿Cuándo usar `Verify`?**
Cuando la llamada es el comportamiento esperado: cobrar, enviar un email, no guardar si algo falló.

**4. ¿Moq puede mockear métodos de extensión?**
No; solo miembros de interfaces o virtuales.

-----

## Práctica

**Ejercicio 1.** Agrega una prueba que verifique que, si el pago se rechaza, se registra una advertencia con el id del pedido. Usa `FakeLogger`.

<details>
<summary>Solución</summary>

Paquete de pruebas: `Microsoft.Extensions.Diagnostics.Testing`.

```csharp
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.Testing;

[Fact]
public async Task PagarAsync_PagoRechazado_RegistraAdvertencia()
{
    var logger = new FakeLogger<ServicioPedidos>();
    var servicio = new ServicioPedidos(_repositorio.Object, _pasarela.Object, logger);
    _pasarela.Setup(p => p.CobrarAsync(7, 120m, It.IsAny<CancellationToken>())).ReturnsAsync(false);

    await servicio.PagarAsync(7);

    FakeLogRecord registro = Assert.Single(logger.Collector.GetSnapshot());
    Assert.Equal(LogLevel.Warning, registro.Level);
    Assert.Equal("Pago rechazado para el pedido 7", registro.Message);
}
```

`FakeLogger` guarda cada log con su nivel, mensaje formateado y propiedades. Es mucho más legible que verificar el método genérico `Log<TState>` de un `Mock<ILogger>`.

</details>

**Ejercicio 2.** Reemplaza el mock de `IRepositorioPedidos` por un fake en memoria y reescribe la prueba del pago aprobado sin `Verify`.

<details>
<summary>Solución</summary>

```csharp
public sealed class RepositorioPedidosEnMemoria : IRepositorioPedidos
{
    private readonly Dictionary<int, Pedido> _pedidos = [];
    public List<int> Guardados { get; } = [];

    public void Agregar(Pedido pedido) => _pedidos[pedido.Id] = pedido;

    public Task<Pedido?> ObtenerAsync(int id, CancellationToken ct) =>
        Task.FromResult(_pedidos.GetValueOrDefault(id));

    public Task GuardarAsync(Pedido pedido, CancellationToken ct)
    {
        _pedidos[pedido.Id] = pedido;
        Guardados.Add(pedido.Id);
        return Task.CompletedTask;
    }
}

[Fact]
public async Task PagarAsync_PagoAprobado_ConRepositorioFake()
{
    var repositorio = new RepositorioPedidosEnMemoria();
    var pedido = new Pedido { Id = 7, Total = 120m };
    repositorio.Agregar(pedido);
    _pasarela.Setup(p => p.CobrarAsync(7, 120m, It.IsAny<CancellationToken>())).ReturnsAsync(true);
    var servicio = new ServicioPedidos(repositorio, _pasarela.Object, NullLogger<ServicioPedidos>.Instance);

    await servicio.PagarAsync(7);

    Assert.Equal(EstadoPedido.Pagado, pedido.Estado);
    Assert.Equal(new[] { 7 }, repositorio.Guardados);
}
```

El fake se escribe una vez y se reutiliza en muchas pruebas; verificas estado (`Guardados`) con asserts normales, en lugar de configurar y verificar expresiones de Moq en cada una.

</details>

-----

## Siguiente lección

[Cobertura de código con Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md)
