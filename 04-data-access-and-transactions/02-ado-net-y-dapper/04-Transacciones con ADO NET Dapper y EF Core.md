# Transacciones con ADO.NET, Dapper y EF Core

## En una frase

Una transacción agrupa varias sentencias para que se confirmen **todas** o **ninguna**; en ADO.NET y Dapper la abres y la pasas a cada comando, en EF Core `SaveChanges` ya es transaccional, y las tres herramientas pueden compartir **una misma** transacción si comparten la conexión.

-----

## Antes de empezar

Conviene que ya sepas:

* Conexiones y comandos: [ADO.NET a fondo](01-ADO%20NET%20a%20fondo.md).
* Dapper sobre la conexión de EF Core: [Dapper avanzado y convivencia con EF Core](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md).
* Unidad de trabajo y `SaveChanges`: [DbContext, entidades y configuración](../01-entity-framework-core/02-DbContext%20entidades%20y%20configuracion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Transacción:** grupo de operaciones que la base confirma o descarta como una unidad.
* **ACID:** atomicidad, consistencia, aislamiento y durabilidad, las cuatro garantías de una transacción.
* **Commit / rollback:** confirmar o deshacer todo lo hecho dentro de la transacción.
* **Nivel de aislamiento:** cuánto ve una transacción de los cambios que otras todavía no confirmaron.
* **Deadlock (interbloqueo):** dos transacciones que esperan, cada una, un bloqueo que tiene la otra; la base cancela una.
* **Estrategia de ejecución:** mecanismo de EF Core que reintenta operaciones ante errores transitorios.
* **Savepoint:** punto intermedio dentro de una transacción al que se puede volver sin deshacer todo.

-----

## El problema

Una transferencia entre cuentas:

```csharp
await conexion.ExecuteAsync("UPDATE Cuentas SET Saldo = Saldo - @Monto WHERE Id = @Origen", datos);
await conexion.ExecuteAsync("UPDATE Cuentas SET Saldo = Saldo + @Monto WHERE Id = @Destino", datos);
```

Entre las dos sentencias, la red se corta, el proceso se reinicia o una regla lanza una excepción. Resultado: Ana tiene 300 pesos menos, Luis no recibió nada y **300 pesos desaparecieron** del sistema.

El equipo agrega una transacción, pero solo en el código de ADO.NET; después alguien agrega el registro del movimiento con EF Core, usando otro `DbContext` con su propia conexión:

```csharp
await using var tx = await conexion.BeginTransactionAsync();
await conexion.ExecuteAsync(debitar, datos, tx);
await conexion.ExecuteAsync(acreditar, datos, tx);

db.Movimientos.Add(new Movimiento { ... });
await db.SaveChangesAsync();      // otra conexión, otra transacción

await tx.CommitAsync();           // si esto falla, el movimiento ya quedó guardado
```

Las consecuencias:

* dinero que aparece o desaparece por fallas a mitad de camino;
* "transacciones" que solo cubren parte del trabajo porque cada herramienta abrió la suya;
* inconsistencias que no se ven en desarrollo, donde nada falla en el momento justo.

-----

## Cómo funciona

### 1. ACID en una tabla

| Garantía | Significa | En la transferencia |
| --- | --- | --- |
| Atomicidad | Todo o nada | Débito y crédito se confirman juntos |
| Consistencia | Las reglas (FK, `CHECK`, únicos) se cumplen al confirmar | `CHECK (Saldo >= 0)` impide saldos negativos |
| Aislamiento | Las transacciones concurrentes no se pisan (según el nivel) | Nadie ve a Ana debitada sin que Luis esté acreditado |
| Durabilidad | Lo confirmado sobrevive a un corte | Tras el commit, el reinicio no deshace nada |

### 2. ADO.NET y Dapper: abrir, pasar, confirmar

```csharp
await using var tx = await conexion.BeginTransactionAsync(ct);
try
{
    await conexion.ExecuteAsync(debitar, datos, transaction: tx);
    await conexion.ExecuteAsync(acreditar, datos, transaction: tx);
    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

* La conexión tiene que estar **abierta** antes de `BeginTransactionAsync`.
* **Cada comando** recibe la transacción (`cmd.Transaction = tx` en ADO.NET, `transaction: tx` en Dapper). Si olvidas uno, SQL Server y SQLite lanzan `InvalidOperationException: Execute requires the command to have a transaction object when the connection assigned to the command is in a pending local transaction`.
* Si la transacción se descarta (`Dispose`) sin `Commit`, se hace rollback. El `catch` explícito sirve para dejar clara la intención y registrar el error.

### 3. EF Core: `SaveChanges` ya es una transacción

```csharp
cuentaAna.Saldo -= 300;
cuentaLuis.Saldo += 300;
db.Movimientos.Add(new Movimiento { ... });
await db.SaveChangesAsync();   // dos UPDATE + un INSERT, todos o ninguno
```

No necesitas `BeginTransaction` si todo el trabajo cabe en **un** `SaveChanges`. Lo necesitas cuando hay varios `SaveChanges`, o cuando mezclas EF Core con SQL propio:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);

await db.Cuentas.Where(c => c.Id == origen).ExecuteUpdateAsync(s => s.SetProperty(c => c.Saldo, c => c.Saldo - monto), ct);
db.Movimientos.Add(new Movimiento { ... });
await db.SaveChangesAsync(ct);

await tx.CommitAsync(ct);
```

`ExecuteUpdateAsync` no pasa por `SaveChanges`: sin la transacción explícita, se confirmaría por separado.

### 4. Compartir una transacción entre EF Core y Dapper

Dos caminos, según quién la abre:

```text
A) La abre EF Core                              B) La abre ADO.NET / Dapper
tx = db.Database.BeginTransactionAsync()        conexion.Open(); tx = conexion.BeginTransactionAsync()
conexion = db.Database.GetDbConnection()        opciones = UseSqlite(conexion)   ← misma conexión
Dapper: transaction: tx.GetDbTransaction()      db.Database.UseTransactionAsync(tx)
db.SaveChangesAsync()                           db.SaveChangesAsync()
tx.CommitAsync()                                tx.CommitAsync()
```

La condición es **la misma conexión**. Una transacción local pertenece a una conexión: dos `DbContext` con sus propias conexiones nunca comparten transacción, aunque usen la misma cadena.

### 5. Niveles de aislamiento

| Nivel | Lecturas sucias | Lecturas no repetibles | Filas fantasma | Nota |
| --- | --- | --- | --- | --- |
| Read Uncommitted | Sí | Sí | Sí | Lee datos que pueden deshacerse |
| Read Committed | No | Sí | Sí | **Por defecto en SQL Server** y PostgreSQL |
| Repeatable Read | No | No | Sí (en SQL Server) | Bloquea lo leído hasta el final |
| Serializable | No | No | No | Como si las transacciones fueran en fila; más bloqueos |
| Snapshot (SQL Server) | No | No | No | Versiones de filas en lugar de bloqueos de lectura |

```csharp
await using var tx = await conexion.BeginTransactionAsync(IsolationLevel.Serializable, ct);
```

Más aislamiento significa más bloqueos y más *deadlocks*. Sube el nivel solo para la operación que lo necesita. SQLite trabaja siempre como serializable.

### 6. Transacciones cortas

Mientras una transacción está abierta, retiene bloqueos. Dentro de ella:

* ❌ llamadas HTTP, envío de emails, esperas al usuario;
* ❌ consultas pesadas que no necesitan formar parte de la unidad atómica;
* ✔ solo las sentencias que deben confirmarse juntas.

Lee y valida **antes**; abre la transacción para escribir; confirma y sal.

### 7. Reintentos y transacciones en EF Core

Con SQL Server en la nube es común activar reintentos: `UseSqlServer(cadena, o => o.EnableRetryOnFailure())`. Entonces esto falla:

```csharp
await using var tx = await db.Database.BeginTransactionAsync();
// InvalidOperationException: The configured execution strategy 'SqlServerRetryingExecutionStrategy'
// does not support user-initiated transactions...
```

Porque si falla a mitad, EF Core no sabe repetir tu bloque entero. La solución es entregarle el bloque completo:

```csharp
var estrategia = db.Database.CreateExecutionStrategy();
await estrategia.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync();
    // ... todo el trabajo ...
    await tx.CommitAsync();
});
```

Ante un error transitorio, la estrategia **vuelve a ejecutar el bloque entero**: no puede tener efectos externos que no se puedan repetir (como un email ya enviado).

### 8. `TransactionScope`: la transacción ambiental

```csharp
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
// conexiones que se abran aquí se enlistan solas
scope.Complete();
```

Es cómodo, pero tiene trampas: sin `TransactionScopeAsyncFlowOption.Enabled`, la transacción no sigue al código después de un `await`; y si dentro se abren dos conexiones a bases distintas, intenta escalar a una transacción **distribuida**, que en .NET moderno solo existe en Windows (desde .NET 7). Para una sola base, prefiere la transacción explícita de la conexión o de EF Core.

-----

## Ejemplo completo

Consola con la transferencia en cuatro versiones. `Transacciones.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Dapper" Version="2.1.35" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using Dapper;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

// Una conexión abierta que comparten Dapper y EF Core
await using var conexion = new SqliteConnection("Data Source=banco.db");
await conexion.OpenAsync();
var opciones = new DbContextOptionsBuilder<BancoContext>().UseSqlite(conexion).Options;

const string Debitar = "UPDATE Cuentas SET Saldo = Saldo - @Monto WHERE Id = @Id";
const string Acreditar = "UPDATE Cuentas SET Saldo = Saldo + @Monto WHERE Id = @Id";

// 1. Sin transacción: la falla a mitad deja el débito hecho
await Reiniciar();
await Mostrar("Inicial");
try
{
    await conexion.ExecuteAsync(Debitar, new { Monto = 300, Id = 1 });
    FallaSimulada();
    await conexion.ExecuteAsync(Acreditar, new { Monto = 300, Id = 2 });
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"  error: {ex.Message}");
}
await Mostrar("Sin transacción");

// 2. Con transacción: la falla deshace el débito
await Reiniciar();
await using (var tx = await conexion.BeginTransactionAsync())
{
    try
    {
        await conexion.ExecuteAsync(Debitar, new { Monto = 300, Id = 1 }, tx);
        FallaSimulada();
        await conexion.ExecuteAsync(Acreditar, new { Monto = 300, Id = 2 }, tx);
        await tx.CommitAsync();
    }
    catch (InvalidOperationException ex)
    {
        await tx.RollbackAsync();
        Console.WriteLine($"  error: {ex.Message} (rollback)");
    }
}
await Mostrar("Con transacción");

// 3. Dapper debita y EF Core acredita y registra, en la MISMA transacción
await Reiniciar();
await using (var tx = await conexion.BeginTransactionAsync())
{
    await conexion.ExecuteAsync(Debitar, new { Monto = 300, Id = 1 }, tx);

    await using var db = new BancoContext(opciones);
    await db.Database.UseTransactionAsync(tx);

    var luis = await db.Cuentas.SingleAsync(c => c.Id == 2);
    luis.Saldo += 300;
    db.Movimientos.Add(new Movimiento { OrigenId = 1, DestinoId = 2, Monto = 300 });
    await db.SaveChangesAsync();

    await tx.CommitAsync();
}
Console.WriteLine($"  commit: movimientos registrados = {await conexion.ExecuteScalarAsync<int>("SELECT COUNT(*) FROM Movimientos")}");
await Mostrar("Dapper + EF Core");

// 4. Olvidar pasar la transacción a un comando
await Reiniciar();
await using (var tx = await conexion.BeginTransactionAsync())
{
    try
    {
        await conexion.ExecuteAsync(Debitar, new { Monto = 300, Id = 1 });   // sin tx
        await tx.CommitAsync();
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"  error: {ex.GetType().Name} (el comando no recibió la transacción)");
    }
}   // Dispose sin Commit: rollback
await Mostrar("Olvidar la tx");

async Task Reiniciar() => await conexion.ExecuteAsync("""
    DROP TABLE IF EXISTS Movimientos;
    DROP TABLE IF EXISTS Cuentas;
    CREATE TABLE Cuentas (
        Id      INTEGER PRIMARY KEY,
        Titular TEXT    NOT NULL,
        Saldo   INTEGER NOT NULL CHECK (Saldo >= 0)
    );
    CREATE TABLE Movimientos (
        Id        INTEGER PRIMARY KEY AUTOINCREMENT,
        OrigenId  INTEGER NOT NULL,
        DestinoId INTEGER NOT NULL,
        Monto     INTEGER NOT NULL
    );
    INSERT INTO Cuentas (Id, Titular, Saldo) VALUES (1, 'Ana', 1000), (2, 'Luis', 500);
    """);

async Task Mostrar(string etiqueta)
{
    var saldos = (await conexion.QueryAsync<long>("SELECT Saldo FROM Cuentas ORDER BY Id")).ToList();
    Console.WriteLine($"{etiqueta,-16} -> Ana {saldos[0]} | Luis {saldos[1]} | total {saldos.Sum()}");
}

static void FallaSimulada() => throw new InvalidOperationException("Falla entre el débito y el crédito");

public class Cuenta
{
    public int Id { get; set; }
    public string Titular { get; set; } = "";
    public int Saldo { get; set; }
}

public class Movimiento
{
    public int Id { get; set; }
    public int OrigenId { get; set; }
    public int DestinoId { get; set; }
    public int Monto { get; set; }
}

public sealed class BancoContext(DbContextOptions<BancoContext> options) : DbContext(options)
{
    public DbSet<Cuenta> Cuentas => Set<Cuenta>();
    public DbSet<Movimiento> Movimientos => Set<Movimiento>();
}
```

Salida:

```text
Inicial          -> Ana 1000 | Luis 500 | total 1500
  error: Falla entre el débito y el crédito
Sin transacción  -> Ana 700 | Luis 500 | total 1200
  error: Falla entre el débito y el crédito (rollback)
Con transacción  -> Ana 1000 | Luis 500 | total 1500
  commit: movimientos registrados = 1
Dapper + EF Core -> Ana 700 | Luis 800 | total 1500
  error: InvalidOperationException (el comando no recibió la transacción)
Olvidar la tx    -> Ana 1000 | Luis 500 | total 1500
```

Qué observar:

* Sin transacción, el total del sistema bajó de 1500 a 1200: el débito se confirmó solo.
* Con transacción, la misma falla dejó todo como estaba. El **total** es la invariante que la transacción protege.
* En el tercer caso, un `UPDATE` de Dapper y el `UPDATE` + `INSERT` de `SaveChanges` se confirmaron juntos porque el contexto usa **la misma conexión** y se unió con `UseTransactionAsync`.
* En el cuarto, el proveedor rechazó un comando sin transacción en una conexión con transacción pendiente; el `Dispose` sin `Commit` hizo rollback.
* `FallaSimulada()` es un método para que el compilador no marque como inalcanzable el código que sigue a un `throw` directo.

-----

## Errores comunes

**1. Varias escrituras relacionadas sin transacción.**
Qué pasa: datos a medio actualizar ante cualquier falla intermedia.
Por qué: cada sentencia se confirma sola (*autocommit*).
Arreglo: una transacción alrededor de lo que debe ser atómico; en EF Core, un solo `SaveChanges` cuando alcance.

**2. No pasar la transacción a cada comando.**
Qué pasa: `InvalidOperationException: Execute requires the command to have a transaction object...`.
Por qué: ADO.NET no asigna la transacción de la conexión a los comandos automáticamente.
Arreglo: `cmd.Transaction = tx` o `transaction: tx` en cada llamada de Dapper.

**3. Creer que dos `DbContext` comparten transacción.**
Qué pasa: uno confirma y el otro no; o el `Commit` de uno no incluye los cambios del otro.
Por qué: cada contexto con su propia conexión tiene su propia transacción.
Arreglo: un solo contexto, o la misma conexión con `UseTransaction`.

**4. Transacciones largas con llamadas externas.**
Qué pasa: bloqueos retenidos, *timeouts* y *deadlocks* bajo carga.
Por qué: los bloqueos duran hasta el `Commit`, y una llamada HTTP puede tardar segundos.
Arreglo: valida y llama afuera; dentro, solo las escrituras. Para efectos externos, outbox.

**5. `BeginTransaction` con reintentos activados.**
Qué pasa: `InvalidOperationException: The configured execution strategy ... does not support user-initiated transactions`.
Por qué: EF Core no puede reintentar un bloque que no controla.
Arreglo: `CreateExecutionStrategy().ExecuteAsync(...)` con todo el bloque adentro.

**6. `TransactionScope` sin flujo asíncrono.**
Qué pasa: `InvalidOperationException: A TransactionScope must be disposed on the same thread that it was created`, o comandos que quedan fuera de la transacción.
Por qué: por defecto, la transacción ambiental no viaja con `await`.
Arreglo: `new TransactionScope(TransactionScopeAsyncFlowOption.Enabled)`, o mejor, transacciones explícitas.

-----

## Según la versión de .NET

* **.NET Framework 1.0:** `IDbTransaction` y `BeginTransaction` en ADO.NET.
* **.NET Framework 2.0:** `System.Transactions` y `TransactionScope`.
* **.NET Framework 4.5.1:** `TransactionScopeAsyncFlowOption` para `async`/`await`.
* **.NET Core 3.0:** `BeginTransactionAsync`, `CommitAsync`, `RollbackAsync` en `DbConnection` y `DbTransaction`.
* **EF Core 5:** savepoints automáticos: si `SaveChanges` falla dentro de tu transacción, solo se deshace ese `SaveChanges`; además, `CreateSavepointAsync` manual.
* **.NET 7:** transacciones distribuidas con `TransactionScope`, solo en Windows.
* **EF Core 7:** `SaveChanges` ya no abre transacción explícita cuando hay una sola sentencia.

-----

## Cuándo sí y cuándo no

**Usa una transacción explícita cuando:** varias escrituras deben ser atómicas y no caben en un solo `SaveChanges`, o mezclas EF Core con Dapper, ADO.NET o `ExecuteUpdate`.

**Sube el nivel de aislamiento cuando:** una operación concreta lee y decide con datos que otra transacción podría cambiar a la vez (por ejemplo, asignar el último cupo), y la concurrencia optimista no alcanza.

**No uses transacciones cuando:** es una sola sentencia (ya es atómica) o un solo `SaveChanges`. Y nunca para "envolver" llamadas a servicios externos: la base no puede deshacer un email ni un cobro en otra API.

-----

## Resumen en 5 líneas

1. Una transacción confirma todo o nada; protege invariantes como el total de dinero del sistema.
2. En ADO.NET y Dapper, cada comando debe recibir la transacción.
3. `SaveChanges` ya es atómico; la transacción explícita es para varios `SaveChanges` o SQL propio.
4. EF Core y Dapper comparten transacción solo si comparten conexión (`GetDbTransaction` o `UseTransaction`).
5. Transacciones cortas, sin llamadas externas; con reintentos, usa la estrategia de ejecución.

-----

## Para profundizar

<details>
<summary>Transacción frente a concurrencia optimista</summary>

Una transacción garantiza atomicidad **dentro** de una operación. No impide que dos usuarios lean el mismo saldo, cada uno reste 300 en memoria y guarden uno después del otro: el segundo pisa al primero (*lost update*). Contra eso hay dos defensas: sentencias que calculan en la base (`SET Saldo = Saldo - @Monto`, como en el ejemplo) o concurrencia optimista con un token de versión, que hace fallar al segundo con `DbUpdateConcurrencyException`. Lo viste en [CRUD seguro con MVC y EF Core](../01-entity-framework-core/07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md).

</details>

<details>
<summary>Deadlocks: por qué ocurren y qué hacer</summary>

La transacción A bloquea la cuenta 1 y quiere la 2; la B bloqueó la 2 y quiere la 1. Ninguna avanza y SQL Server elige una "víctima" que falla con el error 1205. Mitigaciones: acceder a las filas siempre en el **mismo orden** (por ejemplo, por `Id` ascendente), transacciones cortas, índices que eviten bloquear de más, y reintentar la víctima (es un error transitorio que `EnableRetryOnFailure` reintenta).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una transacción hace que varias operaciones se confirmen juntas o se deshagan juntas. En ADO.NET y Dapper la abro con `BeginTransaction`, la paso a cada comando y hago `Commit`; si algo falla, rollback. En EF Core, `SaveChanges` ya es una transacción.

### Respuesta ampliada (semi-senior)

Uso transacciones explícitas cuando el trabajo no cabe en un `SaveChanges` o mezclo EF Core con Dapper o `ExecuteUpdate`. Para compartirla, uso la misma conexión: o `GetDbTransaction()` de la transacción de EF Core para Dapper, o `UseTransaction` en un contexto creado sobre la conexión de ADO.NET. Las mantengo cortas, sin llamadas externas, y con reintentos activados envuelvo el bloque en `CreateExecutionStrategy().ExecuteAsync`. Conozco los niveles de aislamiento (Read Committed por defecto en SQL Server), evito subirlos sin necesidad y distingo atomicidad de concurrencia: contra *lost updates* uso tokens de concurrencia o actualizaciones relativas en SQL.

### Preguntas frecuentes de seguimiento

**1. ¿`SaveChanges` necesita `BeginTransaction`?**
No: cada llamada es atómica. Solo hace falta para agrupar varias llamadas o SQL propio.

**2. ¿Qué pasa si no hago `Commit`?**
Al descartar la transacción se hace rollback.

**3. ¿Dos `DbContext` pueden compartir una transacción?**
Solo si usan la misma conexión y se unen con `UseTransaction`.

**4. ¿Qué nivel de aislamiento usa SQL Server por defecto?**
Read Committed.

-----

## Práctica

**Ejercicio 1.** Reescribe la transferencia solo con EF Core, en un `DbContext`, sin `BeginTransaction`, y explica por qué es atómica.

<details>
<summary>Solución</summary>

```csharp
public async Task TransferirAsync(int origenId, int destinoId, int monto, CancellationToken ct)
{
    var cuentas = await db.Cuentas
        .Where(c => c.Id == origenId || c.Id == destinoId)
        .ToDictionaryAsync(c => c.Id, ct);

    var origen = cuentas[origenId];
    var destino = cuentas[destinoId];
    if (origen.Saldo < monto)
    {
        throw new InvalidOperationException("Saldo insuficiente.");
    }

    origen.Saldo -= monto;
    destino.Saldo += monto;
    db.Movimientos.Add(new Movimiento { OrigenId = origenId, DestinoId = destinoId, Monto = monto });

    await db.SaveChangesAsync(ct);
}
```

Los dos `UPDATE` y el `INSERT` salen en un único `SaveChangesAsync`, que EF Core envuelve en una transacción. Ojo: entre leer y guardar, otra transferencia podría cambiar el saldo de Ana; para evitar el *lost update*, agrega un token de concurrencia a `Cuenta`.

</details>

**Ejercicio 2.** Abre la transacción desde EF Core y haz que una consulta de Dapper participe en ella.

<details>
<summary>Solución</summary>

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);

var conexion = db.Database.GetDbConnection();
await conexion.ExecuteAsync(
    "UPDATE Cuentas SET Saldo = Saldo - @Monto WHERE Id = @Id",
    new { Monto = 300, Id = 1 },
    transaction: tx.GetDbTransaction());

db.Movimientos.Add(new Movimiento { OrigenId = 1, DestinoId = 2, Monto = 300 });
await db.SaveChangesAsync(ct);

await tx.CommitAsync(ct);
```

`GetDbTransaction()` (en `Microsoft.EntityFrameworkCore.Storage`) devuelve la `DbTransaction` de ADO.NET que hay detrás de la transacción de EF Core. Como Dapper usa la conexión del contexto, ambas operaciones quedan en la misma unidad atómica.

</details>

-----

## Siguiente lección

Terminaste el módulo. Vuelve al [índice](README.md). El siguiente paso del bloque es concurrencia optimista e índices en profundidad.
