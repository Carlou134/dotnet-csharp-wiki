# Ejercicios: Clean y Hexagonal Architecture

Requisitos previos: las cuatro lecciones del módulo y la solución `Tienda` de la [lección 3](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md) (Dominio, Aplicación, Infraestructura y Api).

-----

## Ejercicios guiados

### Ejercicio 1. Sacar un caso de uso de un controlador

**Objetivo:** convertir lógica incrustada en un controlador en un caso de uso con puertos.

**Contexto:** es el controlador del problema de la [lección 2](02-Puertos%20y%20adaptadores.md): carga el pedido con EF Core, llama a la pasarela con `HttpClient` y confirma, todo en la acción. Es el punto de partida más común en proyectos reales: no se reescribe todo, se extrae caso por caso.

```csharp
[HttpPost("{id:guid}/pagar")]
public async Task<IActionResult> Pagar(Guid id)
{
    var pedido = await _db.Pedidos.Include(p => p.Lineas).SingleOrDefaultAsync(p => p.Id == id);
    if (pedido is null) return NotFound();
    var r = await _http.PostAsJsonAsync("https://api.pasarela.com/cobros", new { monto = pedido.Total });
    if (!r.IsSuccessStatusCode) return StatusCode(402);
    pedido.Confirmar();
    await _db.SaveChangesAsync();
    return NoContent();
}
```

**Instrucciones:**

1. Define el puerto de salida `IPasarelaPago` en Aplicación.
2. Crea el caso de uso `PagarPedido` que devuelva un resultado con motivo de fallo explícito.
3. Reduce el endpoint a traducir HTTP ↔ caso de uso.

<details>
<summary>Solución</summary>

**Aplicación:**

```csharp
public interface IPasarelaPago
{
    Task<bool> CobrarAsync(Guid pedidoId, decimal monto, CancellationToken ct);
}

public enum MotivoFallo { Ninguno, NoExiste, PagoRechazado }

public sealed record ResultadoPago(MotivoFallo Motivo)
{
    public bool Exito => Motivo == MotivoFallo.Ninguno;
}

public sealed class PagarPedido(IPedidoRepository repositorio, IPasarelaPago pasarela)
{
    public async Task<ResultadoPago> EjecutarAsync(Guid pedidoId, CancellationToken ct)
    {
        Pedido? pedido = await repositorio.ObtenerAsync(pedidoId, ct);
        if (pedido is null) return new(MotivoFallo.NoExiste);

        if (!await pasarela.CobrarAsync(pedido.Id, pedido.Total, ct))
        {
            return new(MotivoFallo.PagoRechazado);
        }

        pedido.Confirmar();   // si está vacío o ya confirmado, lanza DomainException → 422
        await repositorio.GuardarCambiosAsync(ct);
        return new(MotivoFallo.Ninguno);
    }
}
```

**Api:**

```csharp
pedidos.MapPost("/{id:guid}/pagar", async (Guid id, PagarPedido casoDeUso, CancellationToken ct) =>
    (await casoDeUso.EjecutarAsync(id, ct)).Motivo switch
    {
        MotivoFallo.Ninguno => Results.NoContent(),
        MotivoFallo.NoExiste => Results.NotFound(),
        MotivoFallo.PagoRechazado => Results.Problem(title: "Pago rechazado", statusCode: StatusCodes.Status402PaymentRequired),
        _ => Results.StatusCode(StatusCodes.Status500InternalServerError)
    });
```

</details>

**Qué observar:** el `enum MotivoFallo` reemplaza la comparación de textos que se criticaba en la lección 2: el adaptador HTTP traduce cada motivo a un código sin interpretar mensajes. Ojo con un detalle de diseño: aquí se cobra **antes** de verificar que el pedido se puede confirmar; la lección 2 muestra por qué conviene preguntar primero al dominio (`AsegurarQuePuedeConfirmarse`).

-----

### Ejercicio 2. Un adaptador de salida HTTP

**Objetivo:** implementar `IPasarelaPago` en Infraestructura con `HttpClient` tipado, sin que la aplicación sepa nada de HTTP.

**Contexto:** la pasarela real recibe `POST /cobros` con `{ "referencia": "...", "importe": 270.00 }` y responde `200` si aprueba o `402` si rechaza. Cualquier otro código es un error del proveedor.

**Instrucciones:**

1. Crea `PasarelaHttp` (`internal`) en Infraestructura.
2. Regístrala con `AddHttpClient` dentro de `AddInfraestructura`, leyendo la URL base de un parámetro.
3. Traduce las respuestas a `bool` y deja que los errores inesperados se propaguen.

<details>
<summary>Solución</summary>

```csharp
internal sealed class PasarelaHttp(HttpClient http) : IPasarelaPago
{
    public async Task<bool> CobrarAsync(Guid pedidoId, decimal monto, CancellationToken ct)
    {
        using HttpResponseMessage respuesta = await http.PostAsJsonAsync(
            "cobros", new { referencia = pedidoId.ToString("N"), importe = monto }, ct);

        return respuesta.StatusCode switch
        {
            HttpStatusCode.OK => true,
            HttpStatusCode.PaymentRequired => false,
            _ => throw new HttpRequestException($"La pasarela respondió {(int)respuesta.StatusCode}.")
        };
    }
}

// En AddInfraestructura(this IServiceCollection services, string cadenaConexion, Uri urlPasarela)
services.AddHttpClient<IPasarelaPago, PasarelaHttp>(c => c.BaseAddress = urlPasarela);
```

(Requiere el paquete `Microsoft.Extensions.Http` en Infraestructura.)

</details>

**Qué observar:** los nombres del proveedor (`referencia`, `importe`) solo existen en el adaptador: es una pequeña capa anticorrupción. Para reintentos y timeouts, agrega `.AddStandardResilienceHandler()` ([Resiliencia de servicios](../03-resiliencia-de-servicios/README.md)); la aplicación no se entera.

-----

### Ejercicio 3. Probar en dos niveles

**Objetivo:** probar `PagarPedido` con una prueba unitaria (fakes) y el endpoint con una prueba de integración.

**Contexto:** la arquitectura permite elegir el nivel de cada prueba. Las reglas del caso de uso van con fakes, en milisegundos; el cableado HTTP, con `WebApplicationFactory` ([Pruebas de integración](../../05-testing-y-calidad/02-pruebas-de-integracion/README.md)).

**Instrucciones:**

1. Prueba unitaria: pago rechazado → el pedido sigue en `Borrador`.
2. Prueba de integración: con la pasarela reemplazada por una falsa que rechaza, `POST /pedidos/{id}/pagar` responde `402`.

<details>
<summary>Solución</summary>

**Unitaria** (proyecto que referencia a Aplicación):

```csharp
sealed class RepositorioFalso : IPedidoRepository
{
    private readonly Dictionary<Guid, Pedido> _pedidos = [];
    public Task<Pedido?> ObtenerAsync(Guid id, CancellationToken ct) => Task.FromResult(_pedidos.GetValueOrDefault(id));
    public void Agregar(Pedido pedido) => _pedidos[pedido.Id] = pedido;
    public Task GuardarCambiosAsync(CancellationToken ct) => Task.CompletedTask;
}

sealed class PasarelaQueRechaza : IPasarelaPago
{
    public Task<bool> CobrarAsync(Guid pedidoId, decimal monto, CancellationToken ct) => Task.FromResult(false);
}

[Fact]
public async Task Pagar_PagoRechazado_ElPedidoSigueEnBorrador()
{
    var repositorio = new RepositorioFalso();
    var pedido = Pedido.Crear(clienteId: 7);
    pedido.AgregarLinea("Teclado", 150m, 1);
    repositorio.Agregar(pedido);

    ResultadoPago r = await new PagarPedido(repositorio, new PasarelaQueRechaza()).EjecutarAsync(pedido.Id, default);

    Assert.Equal(MotivoFallo.PagoRechazado, r.Motivo);
    Assert.Equal(EstadoPedido.Borrador, pedido.Estado);
}
```

**Integración** (proyecto que referencia a Api):

```csharp
[Fact]
public async Task PostPagar_PasarelaRechaza_Devuelve402()
{
    HttpClient cliente = factory.WithWebHostBuilder(b => b.ConfigureTestServices(s =>
        s.AddSingleton<IPasarelaPago, PasarelaQueRechaza>())).CreateClient();

    var creado = await cliente.PostAsJsonAsync("/pedidos", new { clienteId = 7 });
    Uri pedido = creado.Headers.Location!;
    await cliente.PostAsJsonAsync($"{pedido}/lineas", new { producto = "Teclado", precio = 150, cantidad = 1 });

    HttpResponseMessage respuesta = await cliente.PostAsync($"{pedido}/pagar", content: null);

    Assert.Equal(HttpStatusCode.PaymentRequired, respuesta.StatusCode);
}
```

</details>

**Qué observar:** la prueba unitaria no necesita ASP.NET Core ni EF Core porque el caso de uso solo depende de puertos. La de integración reemplaza **un** adaptador de salida (la pasarela) y deja todo lo demás real: endpoints, casos de uso, dominio y EF Core.

-----

## Retos

### Reto 1. Clean en un solo proyecto

**Misión:** mueve la solución de la lección 3 a un único proyecto `Tienda.Api` con carpetas `Dominio/`, `Aplicacion/`, `Infraestructura/` y `Endpoints/`, y reemplaza la protección de las referencias de proyecto por pruebas de arquitectura basadas en espacios de nombres.

**Pista:** con NetArchTest, `Types.InAssembly(...).That().ResideInNamespace("Tienda.Api.Dominio").ShouldNot().HaveDependencyOnAny(...)`. Ya no puedes usar `internal` para ocultar Infraestructura (todo está en el mismo ensamblado): la regla "los endpoints no usan el `DbContext`" también pasa a ser una prueba.

<details>
<summary>Solución</summary>

```csharp
private static readonly Assembly Api = typeof(Program).Assembly;

[Theory]
[InlineData("Tienda.Api.Dominio", new[] { "Tienda.Api.Aplicacion", "Tienda.Api.Infraestructura", "Tienda.Api.Endpoints", "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore" })]
[InlineData("Tienda.Api.Aplicacion", new[] { "Tienda.Api.Infraestructura", "Tienda.Api.Endpoints", "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore" })]
[InlineData("Tienda.Api.Endpoints", new[] { "Tienda.Api.Infraestructura" })]
public void Capa_NoDependeDeLoProhibido(string capa, string[] prohibidos)
{
    TestResult resultado = Types.InAssembly(Api)
        .That().ResideInNamespace(capa)
        .ShouldNot().HaveDependencyOnAny(prohibidos)
        .GetResult();

    Assert.True(resultado.IsSuccessful, string.Join(", ", resultado.FailingTypeNames ?? []));
}
```

`Program.cs` (la raíz de composición) queda fuera de `Endpoints`, así que puede registrar Infraestructura. Ganas simplicidad (un proyecto, sin `internal` por capa) y pagas con disciplina verificada por pruebas en lugar de por el compilador. Es una opción razonable para equipos chicos.

</details>

-----

### Reto 2. Un puerto de consultas con Dapper

**Misión:** agrega `GET /pedidos?estado=Confirmado` que liste id, cliente, total y cantidad de líneas, sin cargar agregados. Define el puerto en Aplicación e impleméntalo en Infraestructura con Dapper sobre la conexión de EF Core.

**Pista:** `IConsultasPedidos.ListarAsync(string? estado, CancellationToken ct)` devuelve `IReadOnlyList<PedidoResumenDto>`. En la implementación, `db.Database.GetDbConnection()` y una consulta con `COUNT` y `GROUP BY` ([Dapper avanzado y convivencia con EF Core](../../04-data-access-and-transactions/02-ado-net-y-dapper/03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md)).

<details>
<summary>Solución</summary>

**Aplicación:**

```csharp
public sealed record PedidoResumenDto(Guid Id, int ClienteId, string Estado, int CantidadLineas);

public interface IConsultasPedidos
{
    Task<IReadOnlyList<PedidoResumenDto>> ListarAsync(string? estado, CancellationToken ct);
}
```

**Infraestructura:**

```csharp
internal sealed class ConsultasPedidosDapper(TiendaDbContext db) : IConsultasPedidos
{
    public async Task<IReadOnlyList<PedidoResumenDto>> ListarAsync(string? estado, CancellationToken ct)
    {
        const string sql = """
            SELECT p.Id, p.ClienteId, p.Estado, COUNT(l.Id) AS CantidadLineas
            FROM Pedidos p
            LEFT JOIN LineaPedido l ON l.PedidoId = p.Id
            WHERE @Estado IS NULL OR p.Estado = @Estado
            GROUP BY p.Id, p.ClienteId, p.Estado
            ORDER BY p.ClienteId
            """;

        var filas = await db.Database.GetDbConnection().QueryAsync<PedidoResumenDto>(
            new CommandDefinition(sql, new { Estado = estado }, cancellationToken: ct));
        return filas.ToList();
    }
}
// en AddInfraestructura: services.AddScoped<IConsultasPedidos, ConsultasPedidosDapper>();
```

Ojo con dos detalles: el nombre de la tabla de líneas lo decide la convención de EF Core (sin `DbSet` para `LineaPedido`, se llama como la clase, `LineaPedido`), y con SQLite el `Guid` y el `COUNT` llegan como `TEXT` e `Int64`, así que el record con constructor puede necesitar ajustes de tipos (ver "Records y tipos del proveedor" en [Dapper: consultas y comandos](../../04-data-access-and-transactions/02-ado-net-y-dapper/02-Dapper%20consultas%20y%20comandos.md)). El total se omitió a propósito: calcularlo en SQL con `decimal` guardado como texto en SQLite es frágil; con PostgreSQL o SQL Server se agrega `SUM(l.PrecioUnitario * l.Cantidad)`.

</details>

-----

## Checkpoint

1. ¿Qué dice la regla de dependencias y en qué se diferencia de la arquitectura en tres capas?
2. ¿Quién define la interfaz del repositorio y quién la implementa? ¿Por qué ese orden?
3. Nombra dos adaptadores de entrada y dos de salida de un mismo sistema.
4. ¿Cómo mapea EF Core una entidad con constructor privado y una lista privada sin atributos en el dominio?
5. Si las capas son proyectos separados, ¿qué reglas siguen necesitando pruebas de arquitectura?
6. Da tres señales de que un sistema tiene más arquitectura de la que necesita.
