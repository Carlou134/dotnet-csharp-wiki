# Ejercicios: rendimiento y concurrencia

Requisitos previos: las cuatro lecciones del módulo. Los ejercicios usan SQLite con EF Core 10 salvo que se indique otro motor.

-----

## Ejercicios guiados

### Ejercicio 1. Reservar asientos sin sobreventa

**Objetivo:** reservar asientos de una función de cine sin vender dos veces el mismo asiento ni superar la capacidad, bajo concurrencia.

**Contexto:** la primera versión lee la función, comprueba `Vendidos < Capacidad`, suma y guarda. Con dos compras simultáneas por el último asiento, las dos pasan. Es el problema del stock de la [lección 1](01-Concurrencia%20optimista%20a%20fondo.md) con otro disfraz.

**Instrucciones:**

1. Escribe la reserva de `cantidad` entradas con una actualización atómica.
2. Si no hay lugar, devuelve `false` sin modificar nada.
3. Explica por qué no hace falta una transacción explícita.

<details>
<summary>Solución</summary>

```csharp
public static async Task<bool> ReservarAsync(CineContext db, int funcionId, int cantidad, CancellationToken ct)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(cantidad);

    int filas = await db.Funciones
        .Where(f => f.Id == funcionId && f.Vendidos + cantidad <= f.Capacidad)
        .ExecuteUpdateAsync(s => s.SetProperty(f => f.Vendidos, f => f.Vendidos + cantidad), ct);

    return filas == 1;
}
```

Con capacidad 100 y 99 vendidas, dos reservas simultáneas de 1: la primera actualiza una fila (100 vendidas); la segunda evalúa la condición contra 100 y actualiza 0 filas.

</details>

**Qué observar:** no hace falta transacción porque es **una** sentencia, y cada sentencia es atómica. Si además hubiera que insertar una fila en `Reservas`, la reserva y el `UPDATE` irían en la misma transacción, y el `INSERT` solo si el `UPDATE` afectó una fila.

-----

### Ejercicio 2. Diagnosticar y arreglar un listado lento

**Objetivo:** usar el plan de ejecución para elegir el índice correcto.

**Contexto:** el listado "pedidos de un cliente en un rango de fechas, del más nuevo al más viejo" tarda segundos. La consulta:

```sql
SELECT Id, Total, Fecha FROM Pedidos
WHERE ClienteId = 42 AND Fecha >= '2026-01-01'
ORDER BY Fecha DESC
```

Hay índices sobre `ClienteId` y sobre `Fecha` por separado.

**Instrucciones:**

1. Predice qué plan elegirá SQLite con esos índices y verifícalo con `EXPLAIN QUERY PLAN`.
2. Diseña un índice compuesto que resuelva filtro y orden.
3. Vuelve a mirar el plan.

<details>
<summary>Solución</summary>

Con índices separados, el motor elige uno (típicamente `ClienteId`, el más selectivo), busca los pedidos del cliente y después filtra la fecha y **ordena** en memoria: el plan muestra `SEARCH Pedidos USING INDEX IX_Pedidos_ClienteId (ClienteId=?)` seguido de `USE TEMP B-TREE FOR ORDER BY`.

Índice compuesto, con la igualdad primero y el rango y el orden después:

```csharp
modelBuilder.Entity<Pedido>().HasIndex(p => new { p.ClienteId, p.Fecha });
```

Con ese índice, el plan pasa a ser una búsqueda por `(ClienteId=? AND Fecha>?)` sin el paso de ordenamiento temporal: dentro de cada cliente, las fechas ya están ordenadas, y el motor puede recorrerlas en orden inverso. Si además agregas `Total` como columna incluida (SQL Server o PostgreSQL), la consulta queda cubierta y no lee la tabla.

</details>

**Qué observar:** dos índices de una columna no equivalen a uno compuesto. El orden `(ClienteId, Fecha)` importa: `(Fecha, ClienteId)` no permitiría ir directo a los pedidos de un cliente.

-----

### Ejercicio 3. Presupuesto de consultas en una prueba

**Objetivo:** escribir una prueba que falle si un listado introduce un N+1.

**Contexto:** el [interceptor contador](03-Detectar%20consultas%20lentas%20en%20produccion.md) sirve también en pruebas. El listado de categorías con su cantidad de tareas debe resolverse en una sola consulta.

**Instrucciones:**

1. Crea un `DbContext` de prueba con SQLite y el interceptor contador.
2. Siembra datos, reinicia el contador y ejecuta el listado.
3. Verifica que se hizo exactamente una consulta.

<details>
<summary>Solución</summary>

```csharp
[Fact]
public async Task ResumenPorCategoria_HaceUnaSolaConsulta()
{
    var contador = new ContadorDeConsultas();
    var opciones = new DbContextOptionsBuilder<TareasContext>()
        .UseSqlite("Data Source=presupuesto.db")
        .AddInterceptors(contador)
        .Options;

    await using var db = new TareasContext(opciones);
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();
    for (int i = 1; i <= 5; i++)
    {
        var categoria = new Categoria { Nombre = $"Categoría {i}" };
        categoria.Tareas.Add(new Tarea { Titulo = "Tarea" });
        db.Categorias.Add(categoria);
    }
    await db.SaveChangesAsync();
    db.ChangeTracker.Clear();
    contador.Reiniciar();

    var resumen = await ServicioReportes.ResumenPorCategoriaAsync(db, CancellationToken.None);

    Assert.Equal(5, resumen.Count);
    Assert.Equal(1, contador.Total);
}
```

Con `ContadorDeConsultas` como el de la lección 3 más un método `Reiniciar()`. Si alguien reescribe `ResumenPorCategoriaAsync` con un bucle de `CountAsync`, la prueba falla con `Expected: 1, Actual: 6`.

</details>

**Qué observar:** con 5 categorías, un N+1 haría 6 consultas; la prueba no depende del tiempo, así que no es frágil. `ChangeTracker.Clear()` evita que las entidades sembradas, ya rastreadas, oculten consultas por *fix-up*.

-----

## Retos

### Reto 1. Conflicto de concurrencia en una API con ETag

**Misión:** agrega a una Minimal API `GET /productos/{id}` (con `ETag`) y `PUT /productos/{id}` (con `If-Match`), y escribe una prueba de integración que simule a Ana y Luis: los dos leen, Ana guarda, Luis recibe `412`.

**Pista:** reutiliza la solución del ejercicio 1 de la [lección 1](01-Concurrencia%20optimista%20a%20fondo.md) y la fábrica de [pruebas de integración](../../05-testing-y-calidad/02-pruebas-de-integracion/01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md). Lee el `ETag` con `respuesta.Headers.ETag!.Tag`.

<details>
<summary>Solución</summary>

```csharp
[Fact]
public async Task Put_ConEtagViejo_Devuelve412()
{
    HttpClient cliente = factory.CreateClient();

    // Ana y Luis leen la misma versión
    HttpResponseMessage lecturaAna = await cliente.GetAsync("/productos/1");
    HttpResponseMessage lecturaLuis = await cliente.GetAsync("/productos/1");
    string etagAna = lecturaAna.Headers.ETag!.Tag;
    string etagLuis = lecturaLuis.Headers.ETag!.Tag;

    // Ana guarda primero
    var putAna = new HttpRequestMessage(HttpMethod.Put, "/productos/1")
    {
        Content = JsonContent.Create(new { precio = 120, stock = 10 })
    };
    putAna.Headers.IfMatch.Add(new EntityTagHeaderValue(etagAna));
    Assert.Equal(HttpStatusCode.NoContent, (await cliente.SendAsync(putAna)).StatusCode);

    // Luis guarda con la versión vieja
    var putLuis = new HttpRequestMessage(HttpMethod.Put, "/productos/1")
    {
        Content = JsonContent.Create(new { precio = 100, stock = 8 })
    };
    putLuis.Headers.IfMatch.Add(new EntityTagHeaderValue(etagLuis));
    Assert.Equal(HttpStatusCode.PreconditionFailed, (await cliente.SendAsync(putLuis)).StatusCode);
}
```

`EntityTagHeaderValue` está en `System.Net.Http.Headers`. El ETag que devuelve la API debe ir entre comillas (`"\"{version}\""`), como exige el estándar; si no, `EntityTagHeaderValue` lanza una excepción de formato.

</details>

-----

### Reto 2. Feed con cursor compuesto

**Misión:** implementa `GET /tareas?despues={cursor}` ordenado por `FechaCreacion DESC, Id DESC`, con cursor opaco y `siguiente: null` en la última página. Siembra 7 tareas, **tres con la misma fecha**, y verifica que recorriendo páginas de 3 aparecen las 7 exactamente una vez.

**Pista:** usa el cursor del ejercicio 1 de la [lección 4](04-Paginacion%20y%20consultas%20eficientes.md). La prueba recorre `while (siguiente is not null)` acumulando ids, y al final compara con `Assert.Equal(7, ids.Distinct().Count())` y `Assert.Equal(7, ids.Count)`.

<details>
<summary>Solución</summary>

```csharp
[Fact]
public async Task Feed_ConFechasEmpatadas_RecorreTodasUnaVez()
{
    var ids = new List<int>();
    string? siguiente = null;

    do
    {
        string url = siguiente is null ? "/tareas" : $"/tareas?despues={Uri.EscapeDataString(siguiente)}";
        var pagina = await _client.GetFromJsonAsync<PaginaJson>(url);
        ids.AddRange(pagina!.Items.Select(i => i.Id));
        siguiente = pagina.Siguiente;
    }
    while (siguiente is not null);

    Assert.Equal(7, ids.Count);
    Assert.Equal(7, ids.Distinct().Count());
}

private sealed record ItemJson(int Id, string Titulo);
private sealed record PaginaJson(List<ItemJson> Items, string? Siguiente);
```

Si la consulta del endpoint ordenara solo por `FechaCreacion` (sin `ThenByDescending(t => t.Id)` ni el desempate en el `Where`), las tres tareas con la misma fecha podrían repetirse o perderse en el corte entre páginas, y alguna de las dos aserciones fallaría. `Uri.EscapeDataString` hace falta porque el Base64 puede contener `+`, `/` y `=`.

</details>

-----

## Checkpoint

1. ¿Por qué "el cliente gana" con `OriginalValues.SetValues(...)` vuelve a producir una *lost update*?
2. ¿Cómo descontarías stock sin sobreventa y sin transacción explícita?
3. ¿Por qué `WHERE lower(Email) = @email` no usa el índice sobre `Email`, y cómo lo arreglas?
4. Un índice sobre `(A, B)`: ¿sirve para filtrar por `B` sola? ¿Y para `A = 1 ORDER BY B`?
5. ¿Por qué un N+1 puede no aparecer en un reporte de "consultas lentas"?
6. Nombra dos problemas de la paginación por offset que la paginación por cursor resuelve, y una cosa que el cursor no permite.
