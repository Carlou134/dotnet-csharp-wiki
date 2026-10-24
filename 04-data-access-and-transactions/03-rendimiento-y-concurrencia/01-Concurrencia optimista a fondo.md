# Concurrencia optimista a fondo

## En una frase

Cuando dos usuarios modifican la misma fila, alguien pierde su cambio salvo que lo detectes: la concurrencia optimista lo hace con un **token** que se compara al guardar; detectar es la parte fácil, porque decidir **quién gana** (la base, el cliente o una fusión) es una regla de negocio, y para contadores como el stock suele ser mejor una actualización atómica.

-----

## Antes de empezar

Conviene que ya sepas:

* El token de concurrencia básico y `DbUpdateConcurrencyException`: [CRUD seguro con MVC y EF Core](../01-entity-framework-core/07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md).
* Transacciones y niveles de aislamiento: [Transacciones con ADO.NET, Dapper y EF Core](../02-ado-net-y-dapper/04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md).
* `ExecuteUpdateAsync`: [CRUD en APIs con EF Core](../01-entity-framework-core/06-CRUD%20en%20APIs%20con%20EF%20Core.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Actualización perdida (*lost update*):** un cambio que se sobrescribe sin que nadie lo note porque otro guardó encima con datos viejos.
* **Token de concurrencia:** columna que cambia en cada actualización y se usa en el `WHERE` para detectar si la fila cambió desde que se leyó.
* **`rowversion` / `xmin`:** columnas que la base actualiza sola en cada cambio (SQL Server / PostgreSQL).
* **Concurrencia pesimista:** bloquear la fila al leerla para que nadie más la modifique hasta terminar.
* **Actualización atómica:** una sola sentencia que lee y escribe en la base (`SET Stock = Stock - 1 WHERE Stock > 0`), sin ventana entre lectura y escritura.
* **ETag:** encabezado HTTP con la versión de un recurso; con `If-Match`, el cliente pide modificarlo solo si no cambió.

-----

## El problema

Ana y Luis abren el mismo producto en el panel de administración: precio 100, stock 10.

```text
10:00:00  Ana  lee   { precio: 100, stock: 10 }
10:00:05  Luis lee   { precio: 100, stock: 10 }
10:00:30  Ana  guarda precio = 120           → UPDATE Productos SET Precio = 120 ...
10:00:45  Luis guarda stock  = 8             → UPDATE Productos SET Precio = 100, Stock = 8 ...  (su formulario manda todo)
Resultado: { precio: 100, stock: 8 }         ← el aumento de Ana desapareció
```

Nadie recibió un error. El precio volvió a 100 y se vendió a pérdida durante dos días.

El equipo agrega un token de concurrencia, y ahora Luis recibe un error... y la API responde **500**. Lo "arreglan" atrapando la excepción y reintentando con `OriginalValues.SetValues(...)`, como en un ejemplo de internet. Vuelve a pasar lo mismo: el precio de Ana se pierde, ahora con un reintento "exitoso".

En otro endpoint, el checkout descuenta stock así:

```csharp
var producto = await db.Productos.FindAsync(id);
if (producto.Stock < cantidad) return Conflict("Sin stock");
producto.Stock -= cantidad;
await db.SaveChangesAsync();
```

En un evento de ventas, dos compras simultáneas leen stock 1, las dos pasan la validación y el stock termina en **−1**.

Las consecuencias:

* cambios que desaparecen sin aviso;
* conflictos detectados pero resueltos al azar (o como en el ejemplo copiado);
* reglas como "no vender sin stock" rotas por la concurrencia.

-----

## Cómo funciona

### 1. El token: el `WHERE` que detecta el cambio

```sql
UPDATE Productos SET Stock = 8, Version = @nueva
WHERE Id = @id AND Version = @versionQueLeyoLuis
-- 0 filas afectadas → alguien la cambió → DbUpdateConcurrencyException
```

| Motor | Token | Configuración en EF Core |
| --- | --- | --- |
| SQL Server | `rowversion` (8 bytes, lo actualiza la base) | `[Timestamp] public byte[] Version { get; set; }` o `.IsRowVersion()` |
| PostgreSQL | `xmin` (columna de sistema) | `.IsRowVersion()` sobre una propiedad `uint Version` (Npgsql la mapea a `xmin`) |
| SQLite / genérico | Una columna propia | `.IsConcurrencyToken()` y asignar un valor nuevo en cada guardado |

Con `rowversion` y `xmin` no escribes nada: la base cambia el valor en cada `UPDATE`. Con un token propio, tienes que asignarlo tú (por ejemplo, en `SaveChangesAsync`).

### 2. Las tres estrategias de resolución

Al atrapar `DbUpdateConcurrencyException`, `ex.Entries` contiene las entidades en conflicto. Cada una tiene tres juegos de valores:

```text
CurrentValues   lo que Luis quiere guardar      { precio: 100, stock: 8 }
OriginalValues  lo que Luis leyó                 { precio: 100, stock: 10 }
GetDatabaseValuesAsync()  lo que hay ahora       { precio: 120, stock: 10 }
```

| Estrategia | Cómo | Resultado del ejemplo | Cuándo |
| --- | --- | --- | --- |
| **La base gana** | `await entrada.ReloadAsync()` | precio 120, stock 10: se pierde el cambio de Luis | El cambio del usuario no es valioso o puede repetirlo |
| **El cliente gana** | `entrada.OriginalValues.SetValues(valoresDeLaBase)` y volver a guardar | precio 100, stock 8: **se pierde el de Ana** | Casi nunca: es la *lost update* con un paso más |
| **Fusión por propiedad** | Lo que el cliente no tocó, se toma de la base; lo que tocó, se mantiene | precio 120, stock 8 | Cambios en propiedades distintas |
| **Preguntar al usuario** | Devolver `409` con los valores actuales | El usuario decide | Cambios en las mismas propiedades, datos importantes |

¿Por qué "el cliente gana" pisa el precio? Porque `SetValues` cambia los valores originales a los de la base (precio 120), y al guardar, EF Core ve que el valor actual de Luis (100) es distinto del original (120): lo marca como modificado y lo escribe.

### 3. Fusión por propiedad

```csharp
catch (DbUpdateConcurrencyException ex)
{
    EntityEntry entrada = ex.Entries.Single();
    PropertyValues? enLaBase = await entrada.GetDatabaseValuesAsync(ct);
    if (enLaBase is null) { /* la fila se borró: 404 o 410 */ }

    foreach (PropertyEntry propiedad in entrada.Properties)
    {
        if (propiedad.Metadata.IsPrimaryKey()) continue;                 // la clave no se toca
        if (!propiedad.IsModified)
        {
            propiedad.CurrentValue = enLaBase[propiedad.Metadata];   // lo que Luis no tocó: valor actual de la base
        }
        propiedad.OriginalValue = enLaBase[propiedad.Metadata];      // el nuevo "lo que leí" es lo que hay ahora
    }
    await db.SaveChangesAsync(ct);   // puede volver a fallar si alguien guarda otra vez en el medio
}
```

Si los dos modificaron **la misma** propiedad, la fusión automática no puede decidir: eso se le pregunta al usuario o lo define una regla del negocio.

### 4. En una API: ETag e `If-Match`

```text
GET /productos/7          → 200  ETag: "a3f9..."   { precio: 100, stock: 10 }
PUT /productos/7          If-Match: "a3f9..."      { precio: 100, stock: 8 }
    · la versión coincide → 204  ETag: "c71e..."
    · alguien la cambió   → 412 Precondition Failed (o 409 con los valores actuales)
PUT sin If-Match          → 428 Precondition Required (si la API lo exige)
```

El cliente guarda el ETag que recibió y lo devuelve al modificar. La API lo usa como `OriginalValue` del token: así la detección funciona aunque entre el GET y el PUT pasen minutos y cada petición use un `DbContext` distinto.

```csharp
// En el PUT: "lo que el cliente leyó" viene del ETag, no de la base
db.Entry(producto).Property(p => p.Version).OriginalValue = versionDelIfMatch;
```

### 5. Contadores: mejor una sentencia atómica

Para "descontar stock", "sumar puntos" o "incrementar un contador", no hace falta leer y escribir:

```csharp
int filas = await db.Productos
    .Where(p => p.Id == id && p.Stock >= cantidad)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, p => p.Stock - cantidad), ct);

if (filas == 0) return TypedResults.Conflict("Sin stock suficiente.");
```

La condición y el cambio ocurren en **una** sentencia: la base los ejecuta de forma atómica, sin ventana entre la lectura y la escritura. Dos compras simultáneas por la última unidad: una actualiza 1 fila, la otra 0.

### 6. ¿Y la concurrencia pesimista?

```sql
-- PostgreSQL                                  -- SQL Server
SELECT * FROM "Productos" WHERE "Id" = 7       SELECT * FROM Productos WITH (UPDLOCK, ROWLOCK)
FOR UPDATE;                                    WHERE Id = 7;
```

Bloquea la fila dentro de una transacción hasta el `COMMIT`: nadie más puede modificarla mientras tanto.

| | Optimista | Pesimista | Atómica |
| --- | --- | --- | --- |
| Bloquea | No | Sí, hasta el commit | Solo durante la sentencia |
| Conflictos | Se detectan al guardar | Se evitan esperando | No hay ventana |
| Riesgo | Reintentos y resolución | Esperas, *deadlocks*, no sirve entre peticiones HTTP | Solo sirve para cambios expresables en una sentencia |
| Ideal para | Edición de formularios | Operaciones cortas con alta contención dentro de una transacción | Contadores, stock, saldos |

En EF Core, el bloqueo pesimista se escribe con SQL (`FromSql`); no hay API LINQ para eso. Nunca mantengas un bloqueo mientras esperas al usuario.

-----

## Ejemplo completo

Consola con SQLite que reproduce el conflicto de Ana y Luis y lo resuelve con cada estrategia. `Concurrencia.csproj` con `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0. `Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;

var opciones = new DbContextOptionsBuilder<TiendaContext>().UseSqlite("Data Source=concurrencia.db").Options;

await Escenario("El cliente gana", (entrada, enLaBase) =>
{
    entrada.OriginalValues.SetValues(enLaBase);
    return Task.CompletedTask;
});

await Escenario("La base gana", (entrada, enLaBase) => entrada.ReloadAsync());

await Escenario("Fusión por propiedad", (entrada, enLaBase) =>
{
    foreach (PropertyEntry propiedad in entrada.Properties)
    {
        if (propiedad.Metadata.IsPrimaryKey()) continue;
        if (!propiedad.IsModified)
        {
            propiedad.CurrentValue = enLaBase[propiedad.Metadata];
        }
        propiedad.OriginalValue = enLaBase[propiedad.Metadata];
    }
    return Task.CompletedTask;
});

await DescontarStockAtomico();

async Task Escenario(string nombre, Func<EntityEntry, PropertyValues, Task> resolver)
{
    await Reiniciar();
    Console.WriteLine($"== {nombre} ==");

    await using var ana = new TiendaContext(opciones);
    await using var luis = new TiendaContext(opciones);
    Producto deAna = await ana.Productos.SingleAsync();
    Producto deLuis = await luis.Productos.SingleAsync();

    deAna.Precio = 120;
    await ana.SaveChangesAsync();

    deLuis.Stock = 8;
    try
    {
        await luis.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        EntityEntry entrada = ex.Entries.Single();
        PropertyValues enLaBase = (await entrada.GetDatabaseValuesAsync())!;
        Console.WriteLine($"Conflicto: Luis leyó precio {entrada.OriginalValues[nameof(Producto.Precio)]}, la base tiene {enLaBase[nameof(Producto.Precio)]}");

        await resolver(entrada, enLaBase);
        await luis.SaveChangesAsync();
    }

    await using var lectura = new TiendaContext(opciones);
    Producto final = await lectura.Productos.AsNoTracking().SingleAsync();
    Console.WriteLine($"Final: precio {final.Precio}, stock {final.Stock}");
}

async Task DescontarStockAtomico()
{
    await Reiniciar();
    Console.WriteLine("== Stock con actualización atómica ==");

    await using var db = new TiendaContext(opciones);
    foreach (int cantidad in new[] { 6, 3, 2 })
    {
        int filas = await db.Productos
            .Where(p => p.Stock >= cantidad)
            .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, p => p.Stock - cantidad));
        Console.WriteLine($"Descontar {cantidad}: {(filas == 1 ? "ok" : "sin stock suficiente")}");
    }
    Console.WriteLine($"Stock final: {(await db.Productos.AsNoTracking().SingleAsync()).Stock}");
}

async Task Reiniciar()
{
    await using var db = new TiendaContext(opciones);
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();
    db.Productos.Add(new Producto { Nombre = "Teclado", Precio = 100, Stock = 10 });
    await db.SaveChangesAsync();
}

public class Producto
{
    public int Id { get; set; }
    public required string Nombre { get; set; }
    public int Precio { get; set; }   // en pesos enteros, para que la salida no dependa de cómo SQLite guarda decimal
    public int Stock { get; set; }
    public Guid Version { get; set; }
}

public sealed class TiendaContext(DbContextOptions<TiendaContext> options) : DbContext(options)
{
    public DbSet<Producto> Productos => Set<Producto>();

    protected override void OnModelCreating(ModelBuilder modelBuilder) =>
        modelBuilder.Entity<Producto>().Property(p => p.Version).IsConcurrencyToken();

    // SQLite no tiene rowversion: asignamos una versión nueva en cada inserción o modificación
    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entrada in ChangeTracker.Entries<Producto>()
                     .Where(e => e.State is EntityState.Added or EntityState.Modified))
        {
            entrada.Entity.Version = Guid.NewGuid();
        }
        return base.SaveChangesAsync(cancellationToken);
    }
}
```

Salida:

```text
== El cliente gana ==
Conflicto: Luis leyó precio 100, la base tiene 120
Final: precio 100, stock 8
== La base gana ==
Conflicto: Luis leyó precio 100, la base tiene 120
Final: precio 120, stock 10
== Fusión por propiedad ==
Conflicto: Luis leyó precio 100, la base tiene 120
Final: precio 120, stock 8
== Stock con actualización atómica ==
Descontar 6: ok
Descontar 3: ok
Descontar 2: sin stock suficiente
Stock final: 1
```

Qué observar:

* **El cliente gana** detectó el conflicto y aun así perdió el precio de Ana: es la *lost update* con pasos extra. Por eso la "solución" copiada del problema no resolvía nada.
* **La base gana** conserva a Ana y descarta a Luis: él tendría que volver a cargar su stock.
* **La fusión** conserva los dos cambios porque tocaron propiedades distintas.
* La actualización atómica rechazó el tercer descuento sin leer el stock en C#: la condición `Stock >= cantidad` la evaluó la base en la misma sentencia.
* El token es un `Guid` asignado en `SaveChangesAsync` porque SQLite no tiene `rowversion`; en SQL Server o PostgreSQL lo haría la base.

-----

## Errores comunes

**1. Atrapar el conflicto y forzar el guardado ("el cliente gana" por defecto).**
Qué pasa: la *lost update* sigue ocurriendo, ahora con un reintento que parece exitoso.
Por qué: `SetValues` con los valores de la base hace que todo lo distinto del cliente se escriba.
Arreglo: elige una estrategia según el negocio: fusión por propiedad, la base gana o preguntar al usuario.

**2. Responder `500` ante un conflicto.**
Qué pasa: el cliente cree que el servidor falló y reintenta a ciegas.
Por qué: `DbUpdateConcurrencyException` no se maneja.
Arreglo: `409 Conflict` (o `412` con ETag) con los valores actuales, para que el usuario decida.

**3. Leer, validar y escribir contadores en C#.**
Qué pasa: stock negativo, cupos sobrevendidos bajo carga.
Por qué: entre la lectura y el `UPDATE` otra petición hizo lo mismo.
Arreglo: `ExecuteUpdateAsync` con la condición en el `WHERE`, o un token de concurrencia con reintento.

**4. El token de concurrencia en la entidad, pero no en el contrato HTTP.**
Qué pasa: el PUT carga la entidad fresca de la base y la versión siempre coincide: nunca hay conflicto aunque el cliente editara datos viejos.
Por qué: la versión que el cliente leyó no viaja en la petición.
Arreglo: devolver la versión (ETag o campo `version`) en el GET y exigirla en el PUT como valor original.

**5. Bloqueo pesimista que abarca la interacción del usuario.**
Qué pasa: filas bloqueadas durante minutos, esperas y *timeouts* en cascada.
Por qué: se abrió la transacción con `FOR UPDATE` al mostrar el formulario.
Arreglo: el bloqueo pesimista solo dentro de una operación corta del servidor; para formularios, optimista.

-----

## Según la versión de .NET

* **EF Core 1.0:** `IsConcurrencyToken`, `IsRowVersion`, `[Timestamp]` y `DbUpdateConcurrencyException` con `Entries`.
* **EF Core 3.0:** `GetDatabaseValuesAsync` y la API de `PropertyValues` estables para resolver conflictos.
* **EF Core 7:** `ExecuteUpdate` y `ExecuteDelete`, la base de las actualizaciones atómicas sin SQL a mano.
* **Npgsql EF Core 7+:** `IsRowVersion()` sobre una propiedad `uint` se mapea a la columna de sistema `xmin`.
* **ASP.NET Core:** soporte de `ETag`/`If-Match` en respuestas y resultados de archivos; en APIs JSON se implementa a mano o con middleware.

-----

## Cuándo sí y cuándo no

**Usa concurrencia optimista cuando:** varias personas editan los mismos registros y perder un cambio importa (precios, datos de clientes, configuraciones).

**Usa actualizaciones atómicas cuando:** el cambio es un cálculo sobre el valor actual con una condición (stock, saldos, contadores, cupos).

**Usa bloqueo pesimista cuando:** dentro de una operación corta del servidor hay muchísima contención sobre pocas filas y reintentar es caro.

**No uses nada cuando:** el dato lo modifica un solo dueño o "gana el último" es aceptable para el negocio (por ejemplo, la preferencia de tema de un usuario).

-----

## Resumen en 5 líneas

1. Sin control, el último en guardar pisa al anterior sin aviso.
2. Un token de concurrencia en el `WHERE` detecta que la fila cambió: `DbUpdateConcurrencyException`.
3. Resolver es negocio: la base gana, fusión por propiedad o preguntar; "el cliente gana" es la *lost update* de nuevo.
4. En APIs, la versión viaja con ETag/`If-Match` y el conflicto responde `409` o `412`.
5. Para contadores, una actualización atómica con la condición en el `WHERE`.

-----

## Para profundizar

<details>
<summary>Reintentos automáticos con optimista</summary>

Para operaciones del servidor sin intervención humana (por ejemplo, recalcular un total), un bucle de reintento es razonable: atrapar `DbUpdateConcurrencyException`, recargar la entidad (`ReloadAsync`), volver a aplicar la operación sobre los datos frescos y guardar, con un máximo de intentos. Es la base de cómo los *event stores* y los agregados de DDD manejan la contención. Para operaciones con datos que escribió un usuario, no reintentes en silencio: devuelve el conflicto.

</details>

<details>
<summary>Concurrencia y agregados</summary>

En DDD, el token de concurrencia va en la **raíz** del agregado. Si alguien agrega una línea a un pedido, la versión del `Pedido` debe cambiar aunque solo se haya insertado una fila en `LineasPedido`; si no, dos usuarios pueden agregar líneas que juntas violan una invariante (por ejemplo, un máximo de 10 líneas). Se resuelve forzando la modificación de la raíz en cada cambio del agregado (por ejemplo, actualizando una propiedad `ModificadoEn` o la versión en `SaveChangesAsync`).

</details>

-----

## En entrevista

### Respuesta corta (junior)

La concurrencia optimista evita que un usuario pise los cambios de otro. Se agrega un token, como `rowversion`, y EF Core lo usa en el `WHERE` del `UPDATE`; si la fila cambió, lanza `DbUpdateConcurrencyException` y la API responde `409`.

### Respuesta ampliada (semi-senior)

Detecto conflictos con `rowversion` en SQL Server, `xmin` en PostgreSQL o un token propio, y en APIs hago viajar la versión con ETag e `If-Match` para que el valor original sea el que el cliente leyó. Al resolver, sé que "el cliente gana" con `SetValues` vuelve a perder el cambio ajeno; uso fusión por propiedad cuando se tocaron campos distintos, "la base gana" cuando el cambio es repetible, o devuelvo `409` con los valores actuales. Para stock y contadores prefiero una actualización atómica con la condición en el `WHERE` (`ExecuteUpdate`), y reservo el bloqueo pesimista para operaciones cortas con alta contención, nunca abarcando la interacción del usuario.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es una *lost update*?**
Un cambio sobrescrito por otro que guardó encima con datos leídos antes.

**2. ¿Por qué "el cliente gana" no soluciona nada?**
Porque escribe todos sus valores distintos de los actuales, incluidos los que no tocó.

**3. ¿Cómo descontarías stock sin sobreventa?**
Con `UPDATE ... SET Stock = Stock - @n WHERE Id = @id AND Stock >= @n` y comprobando las filas afectadas.

**4. ¿Optimista o pesimista?**
Optimista para edición por usuarios; pesimista solo en operaciones cortas del servidor con mucha contención.

-----

## Práctica

**Ejercicio 1.** Escribe el `PUT /productos/{id}` de una Minimal API que use la versión como ETag: exige `If-Match`, responde `412` si no coincide y devuelve el ETag nuevo.

<details>
<summary>Solución</summary>

```csharp
app.MapPut("/productos/{id:int}", async (int id, ActualizarProducto request, HttpContext http, TiendaContext db, CancellationToken ct) =>
{
    if (!Guid.TryParse(http.Request.Headers.IfMatch.ToString().Trim('"'), out Guid versionLeida))
    {
        return Results.StatusCode(StatusCodes.Status428PreconditionRequired);
    }

    Producto? producto = await db.Productos.FindAsync([id], ct);
    if (producto is null) return Results.NotFound();

    db.Entry(producto).Property(p => p.Version).OriginalValue = versionLeida;   // lo que el cliente leyó
    producto.Precio = request.Precio;
    producto.Stock = request.Stock;

    try
    {
        await db.SaveChangesAsync(ct);
    }
    catch (DbUpdateConcurrencyException)
    {
        return Results.StatusCode(StatusCodes.Status412PreconditionFailed);
    }

    http.Response.Headers.ETag = $"\"{producto.Version}\"";
    return Results.NoContent();
});

public sealed record ActualizarProducto(int Precio, int Stock);
```

El GET correspondiente devuelve `ETag: "{producto.Version}"`. Fijar el `OriginalValue` del token con la versión del cliente es la clave: sin eso, la entidad recién cargada siempre tendría la versión actual y nunca habría conflicto.

</details>

**Ejercicio 2.** Un cupo de inscripción a un curso tiene `CupoMaximo` e `Inscriptos`. Escribe la inscripción para que nunca supere el cupo bajo concurrencia, sin transacción explícita.

<details>
<summary>Solución</summary>

```csharp
int filas = await db.Cursos
    .Where(c => c.Id == cursoId && c.Inscriptos < c.CupoMaximo)
    .ExecuteUpdateAsync(s => s.SetProperty(c => c.Inscriptos, c => c.Inscriptos + 1), ct);

return filas == 1 ? TypedResults.NoContent() : TypedResults.Conflict("No hay cupo.");
```

La condición `Inscriptos < CupoMaximo` y el incremento ocurren en la misma sentencia, así que dos inscripciones simultáneas por el último lugar no pueden pasar ambas. Si además hay que crear la fila de la inscripción, va en la misma transacción que este `UPDATE`, y solo si el `UPDATE` afectó una fila.

</details>

-----

## Siguiente lección

[Índices y planes de ejecución](02-Indices%20y%20planes%20de%20ejecucion.md)
