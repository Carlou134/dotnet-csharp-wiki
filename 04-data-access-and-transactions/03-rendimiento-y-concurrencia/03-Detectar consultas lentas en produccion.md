# Detectar consultas lentas en producción

## En una frase

Las consultas lentas y los N+1 casi nunca aparecen en desarrollo, así que hay que **verlos en producción**: logs de comandos con su duración, un interceptor que avise de las lentas, etiquetas (`TagWith`) que digan de dónde viene cada SQL, trazas con OpenTelemetry y, del lado de la base, las estadísticas de consultas del motor.

-----

## Antes de empezar

Conviene que ya sepas:

* N+1, `Include` y proyecciones: [Relaciones y carga de datos](../01-entity-framework-core/05-Relaciones%20y%20carga%20de%20datos.md).
* Planes de ejecución: [Índices y planes de ejecución](02-Indices%20y%20planes%20de%20ejecucion.md).
* Logs por categoría y nivel: [Logging con ILogger](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/04-Logging%20con%20ILogger.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Interceptor de comandos:** clase de EF Core (`DbCommandInterceptor`) que se ejecuta antes y después de cada comando SQL.
* **Etiqueta de consulta (*query tag*):** comentario que EF Core agrega al SQL con `TagWith` para identificar su origen.
* **Umbral de lentitud:** duración a partir de la cual una consulta se considera lenta y se registra.
* **Query Store / `pg_stat_statements`:** registros del propio motor (SQL Server / PostgreSQL) con estadísticas acumuladas por consulta.
* **Presupuesto de consultas:** máximo de consultas aceptable para una petición; superarlo es una señal de N+1.

-----

## El problema

Un endpoint de reportes tarda 6 segundos, solo en producción y solo a ciertas horas. En desarrollo tarda 80 ms. El equipo:

1. Activa en producción el log de EF Core en `Information` "para ver qué pasa". Se generan 40 GB de logs por día, con **valores de parámetros** (emails, documentos) porque alguien dejó `EnableSensitiveDataLogging()` activo.
2. En medio de ese ruido, encuentra cientos de consultas iguales:

```sql
SELECT COUNT(*) FROM "Tareas" WHERE "CategoriaId" = @p0
```

¿De qué endpoint vienen? El SQL no lo dice. Hay seis lugares en el código que cuentan tareas por categoría.

3. Cuando por fin lo encuentran, es un N+1: un bucle con una consulta por categoría. En desarrollo hay 5 categorías; en producción, 1200.

Las consecuencias:

* problemas de rendimiento invisibles hasta que llegan los usuarios;
* logs gigantes, caros y con datos sensibles;
* horas buscando qué parte del código generó un SQL anónimo.

-----

## Cómo funciona

### 1. Logs de comandos: con criterio

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
  }
}
```

* En `Information`, EF Core registra cada comando: `Executed DbCommand (12ms) [Parameters=[...]] SELECT ...`. Útil en desarrollo; demasiado volumen en producción.
* En `Warning`, solo quedan las advertencias y los errores.
* `EnableSensitiveDataLogging()` agrega los valores de los parámetros: **solo en desarrollo**.

Para producción, en lugar de registrar todo, registra solo lo lento.

### 2. Un interceptor que avisa de las consultas lentas

```csharp
public sealed class ConsultasLentasInterceptor(ILogger<ConsultasLentasInterceptor> logger, TimeSpan umbral)
    : DbCommandInterceptor
{
    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command, CommandExecutedEventData eventData, DbDataReader result, CancellationToken ct = default)
    {
        if (eventData.Duration >= umbral)
        {
            logger.LogWarning("Consulta lenta ({Ms} ms): {Sql}", eventData.Duration.TotalMilliseconds, command.CommandText);
        }
        return base.ReaderExecutedAsync(command, eventData, result, ct);
    }
}

builder.Services.AddSingleton(sp => new ConsultasLentasInterceptor(
    sp.GetRequiredService<ILogger<ConsultasLentasInterceptor>>(), TimeSpan.FromMilliseconds(500)));
builder.Services.AddDbContext<AppDbContext>((sp, o) => o
    .UseNpgsql(cadena)
    .AddInterceptors(sp.GetRequiredService<ConsultasLentasInterceptor>()));
```

* `CommandExecutedEventData.Duration` trae el tiempo que tardó el comando.
* El interceptor es *singleton* y se agrega en el lambda de `AddDbContext`, que recibe el `IServiceProvider`.
* Registrar el SQL **sin** parámetros evita filtrar datos; para reproducir el caso, el SQL y la etiqueta suelen alcanzar.

### 3. Etiquetas: que el SQL diga de dónde viene

```csharp
var resumen = await db.Categorias
    .TagWith("Reportes: resumen por categoría")
    .Select(c => new { c.Nombre, Total = c.Tareas.Count })
    .ToListAsync(ct);
```

```sql
-- Reportes: resumen por categoría

SELECT "c"."Nombre", (SELECT COUNT(*) FROM "Tareas" ...) AS "Total"
FROM "Categorias" AS "c"
```

La etiqueta viaja **dentro del SQL** como comentario: aparece en los logs de EF Core, en el interceptor, en el Query Store de SQL Server, en `pg_stat_statements` y en las herramientas del DBA. `TagWithCallSite()` (EF Core 6+) agrega automáticamente el archivo y la línea del código que generó la consulta.

### 4. Presupuesto de consultas por petición

Un N+1 se reconoce por **cuántas** consultas hace una petición, no por lo que tarda cada una:

```csharp
// Scoped: un contador por petición
public sealed class ContadorDeConsultas : DbCommandInterceptor
{
    public int Total { get; private set; }

    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(
        DbCommand command, CommandEventData eventData, InterceptionResult<DbDataReader> result, CancellationToken ct = default)
    {
        Total++;
        return base.ReaderExecutingAsync(command, eventData, result, ct);
    }
}

builder.Services.AddScoped<ContadorDeConsultas>();
builder.Services.AddDbContext<AppDbContext>((sp, o) => o
    .UseNpgsql(cadena)
    .AddInterceptors(sp.GetRequiredService<ContadorDeConsultas>()));   // sp es el scope de la petición

// Middleware: avisar si una petición supera el presupuesto
app.Use(async (context, next) =>
{
    await next(context);
    var contador = context.RequestServices.GetRequiredService<ContadorDeConsultas>();
    if (contador.Total > 20)
    {
        app.Logger.LogWarning("{Ruta} hizo {Total} consultas", context.Request.Path, contador.Total);
    }
});
```

La misma idea sirve en **pruebas de integración**: una prueba que llama a un endpoint y verifica que hizo como mucho N consultas detecta un N+1 antes de que llegue a producción.

### 5. Advertencias de EF Core que conviene escuchar

```csharp
options.ConfigureWarnings(w => w
    .Throw(RelationalEventId.MultipleCollectionIncludeWarning)   // explosión cartesiana
    .Throw(CoreEventId.RowLimitingOperationWithoutOrderByWarning));
```

En desarrollo y en las pruebas, convertir esas advertencias en excepciones hace que los problemas aparezcan antes del *code review*.

### 6. Trazas y estadísticas del motor

| Herramienta | Qué muestra | Dónde |
| --- | --- | --- |
| OpenTelemetry (`OpenTelemetry.Instrumentation.EntityFrameworkCore`, o la instrumentación del driver: SqlClient / Npgsql) | Cada consulta como un *span* dentro de la traza de la petición, con su duración | Jaeger, Grafana Tempo, Application Insights |
| Query Store (SQL Server) | Consultas más costosas, regresiones de plan, historial | La base de datos |
| `pg_stat_statements` (PostgreSQL) | Tiempo total, llamadas y filas por consulta normalizada | La base de datos |
| MiniProfiler | Consultas por petición en desarrollo, en la propia página | El navegador |

La traza responde "¿por qué esta petición tardó 6 segundos?" (por ejemplo, 1200 *spans* de SQL de 5 ms cada uno). Las estadísticas del motor responden "¿qué consultas consumen más en total?" (una consulta de 5 ms ejecutada un millón de veces puede costar más que una de 3 segundos ejecutada una vez). Ver [Trazas distribuidas](../../02-architecture-and-design-patterns/04-observabilidad-con-opentelemetry/01-Trazas%20distribuidas.md).

-----

## Ejemplo completo

Consola que monitorea las consultas con un interceptor y etiquetas, y compara un dashboard con N+1 con su versión proyectada. `Monitoreo.csproj` con `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0. `Program.cs`:

```csharp
using System.Data.Common;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;

var monitor = new MonitorDeConsultas(umbralLentitud: TimeSpan.FromMilliseconds(500));
var opciones = new DbContextOptionsBuilder<TareasContext>()
    .UseSqlite("Data Source=monitoreo.db")
    .AddInterceptors(monitor)
    .Options;

await using (var db = new TareasContext(opciones))
{
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();
    foreach (string nombre in new[] { "Hogar", "Personal", "Trabajo" })
    {
        var categoria = new Categoria { Nombre = nombre };
        for (int i = 1; i <= 4; i++)
        {
            categoria.Tareas.Add(new Tarea { Titulo = $"{nombre} {i}" });
        }
        db.Categorias.Add(categoria);
    }
    await db.SaveChangesAsync();
}

// Dashboard v1: una consulta por categoría (N+1)
await using (var db = new TareasContext(opciones))
{
    monitor.Reiniciar();
    var categorias = await db.Categorias.TagWith("Dashboard v1: categorías")
        .AsNoTracking().OrderBy(c => c.Nombre).ToListAsync();
    foreach (var categoria in categorias)
    {
        await db.Tareas.TagWith("Dashboard v1: conteo por categoría")
            .CountAsync(t => t.CategoriaId == categoria.Id);
    }
    monitor.Imprimir("Dashboard v1");
}

// Dashboard v2: una proyección
await using (var db = new TareasContext(opciones))
{
    monitor.Reiniciar();
    await db.Categorias.TagWith("Dashboard v2: resumen")
        .OrderBy(c => c.Nombre)
        .Select(c => new { c.Nombre, Total = c.Tareas.Count })
        .ToListAsync();
    monitor.Imprimir("Dashboard v2");
}

Console.WriteLine($"Consultas más lentas que 500 ms: {monitor.Lentas}");

public sealed class MonitorDeConsultas(TimeSpan umbralLentitud) : DbCommandInterceptor
{
    private readonly List<string> _etiquetas = [];
    public int Lentas { get; private set; }

    public void Reiniciar() => _etiquetas.Clear();

    public void Imprimir(string titulo)
    {
        Console.WriteLine($"{titulo} -> consultas: {_etiquetas.Count}");
        foreach (var grupo in _etiquetas.GroupBy(e => e))
        {
            Console.WriteLine($"  {grupo.Count()}x {grupo.Key}");
        }
    }

    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command, CommandExecutedEventData eventData, DbDataReader result, CancellationToken cancellationToken = default)
    {
        _etiquetas.Add(ObtenerEtiqueta(command.CommandText));
        if (eventData.Duration >= umbralLentitud)
        {
            Lentas++;
        }
        return base.ReaderExecutedAsync(command, eventData, result, cancellationToken);
    }

    // EF Core escribe las etiquetas como comentarios "-- ..." al principio del SQL
    private static string ObtenerEtiqueta(string sql) =>
        sql.Split('\n').FirstOrDefault(l => l.StartsWith("-- "))?[3..].Trim() ?? "(sin etiqueta)";
}

public class Categoria
{
    public int Id { get; set; }
    public required string Nombre { get; set; }
    public List<Tarea> Tareas { get; } = [];
}

public class Tarea
{
    public int Id { get; set; }
    public required string Titulo { get; set; }
    public int CategoriaId { get; set; }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Categoria> Categorias => Set<Categoria>();
    public DbSet<Tarea> Tareas => Set<Tarea>();
}
```

Salida:

```text
Dashboard v1 -> consultas: 4
  1x Dashboard v1: categorías
  3x Dashboard v1: conteo por categoría
Dashboard v2 -> consultas: 1
  1x Dashboard v2: resumen
Consultas más lentas que 500 ms: 0
```

Qué observar:

* El monitor no tuvo que entender el SQL: la etiqueta de cada consulta dice qué parte del código la generó. En el problema, el SQL anónimo de "contar tareas" habría aparecido como "Dashboard v1: conteo por categoría", 1200 veces.
* El N+1 se ve como **cantidad** (1 + 3), no como tiempo: cada consulta es rápida y ninguna supera el umbral de lentitud. Por eso hacen falta las dos métricas: duración y cantidad por operación.
* La versión proyectada resuelve todo en una consulta con el `COUNT` calculado en SQL.
* El monitor se reinicia después de sembrar los datos, así que no cuenta los `INSERT`.
* En una aplicación real, en lugar de imprimir, el interceptor registraría con `ILogger` o emitiría métricas; y el contador por petición sería scoped.

-----

## Errores comunes

**1. Log de comandos en `Information` en producción.**
Qué pasa: volúmenes enormes de logs y costos de almacenamiento.
Por qué: se registra cada comando de cada petición.
Arreglo: `Warning` para `Microsoft.EntityFrameworkCore.Database.Command` en producción y un interceptor que registre solo lo lento.

**2. `EnableSensitiveDataLogging` fuera de desarrollo.**
Qué pasa: valores de parámetros (datos personales, tokens) en los logs.
Por qué: la opción quedó activa en la configuración compartida.
Arreglo: activarla solo con `if (builder.Environment.IsDevelopment())`.

**3. Buscar el N+1 por tiempo.**
Qué pasa: ninguna consulta supera el umbral y el problema no aparece en el reporte de "lentas".
Por qué: el costo está en la cantidad, no en cada consulta.
Arreglo: contar consultas por petición (presupuesto) y verlas en las trazas.

**4. SQL sin etiquetas.**
Qué pasa: horas para encontrar qué código generó una consulta problemática.
Por qué: el SQL generado no dice nada de su origen.
Arreglo: `TagWith` en las consultas importantes, o `TagWithCallSite()`.

**5. Optimizar sin mirar las estadísticas del motor.**
Qué pasa: se optimiza la consulta más lenta y la carga total no baja.
Por qué: la que más consume en total puede ser una rápida que se ejecuta millones de veces.
Arreglo: ordenar por tiempo total en el Query Store o en `pg_stat_statements`.

-----

## Según la versión de .NET

* **EF Core 2.2:** `TagWith` para etiquetar consultas.
* **EF Core 3.0:** interceptores (`DbCommandInterceptor`, `DbConnectionInterceptor`, `DbTransactionInterceptor`).
* **EF Core 5:** `ToQueryString()` y logging simple con `LogTo`.
* **EF Core 6:** `TagWithCallSite()`.
* **EF Core 7+:** más eventos en interceptores (materialización, consultas) y mejoras en las advertencias.
* **.NET 8:** métricas y trazas integradas con OpenTelemetry en ASP.NET Core; instrumentaciones de SqlClient y Npgsql maduras.

-----

## Cuándo sí y cuándo no

**Monitorea en producción cuando:** siempre que haya usuarios. Un interceptor de consultas lentas y trazas con OpenTelemetry cuestan poco y evitan diagnósticos a ciegas.

**Usa el presupuesto de consultas cuando:** quieras detectar N+1 de forma sistemática, en pruebas de integración o como alerta en producción.

**No actives el log completo de comandos cuando:** estás en producción; actívalo de forma temporal y acotada (una categoría, un tiempo limitado) si hace falta investigar algo puntual.

-----

## Resumen en 5 líneas

1. Los problemas de rendimiento se ven en producción: hay que instrumentar, no adivinar.
2. Log de comandos en `Warning` y un interceptor que registre solo las consultas lentas, sin parámetros.
3. `TagWith` hace que cada SQL diga de dónde viene, en logs, trazas y estadísticas del motor.
4. El N+1 se detecta contando consultas por petición, no midiendo cada una.
5. Trazas para "por qué tardó esta petición"; Query Store o `pg_stat_statements` para "qué consume más en total".

-----

## Para profundizar

<details>
<summary>Prueba de integración con presupuesto de consultas</summary>

Con `WebApplicationFactory`, registra el interceptor contador en `ConfigureTestServices` (como singleton, con un contador que la prueba pueda leer), llama al endpoint y verifica `Assert.InRange(contador.Total, 1, 3)`. Si alguien introduce un N+1 en ese endpoint, la prueba falla con el número exacto de consultas. Combínalo con la base real de [Testcontainers](../../05-testing-y-calidad/02-pruebas-de-integracion/03-Base%20de%20datos%20real%20con%20Testcontainers.md) para que el SQL sea el de producción.

</details>

<details>
<summary>Consultas lentas por bloqueos, no por el plan</summary>

Una consulta con un plan perfecto puede tardar segundos si está **esperando un bloqueo** de otra transacción. En SQL Server, `sys.dm_exec_requests` muestra qué sesión bloquea a cuál; en PostgreSQL, `pg_stat_activity` y `pg_locks`. Si el interceptor reporta lentitudes intermitentes en consultas simples, revisa transacciones largas antes que índices ([Transacciones](../02-ado-net-y-dapper/04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md)).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para encontrar consultas lentas activo los logs de EF Core con la duración de cada comando y uso `TagWith` para saber de dónde viene cada consulta. Para el N+1, cuento cuántas consultas hace un endpoint.

### Respuesta ampliada (semi-senior)

En producción dejo el log de comandos en `Warning` y uso un `DbCommandInterceptor` que registra las consultas que superan un umbral, sin parámetros sensibles. Etiqueto las consultas importantes con `TagWith` (o `TagWithCallSite`) para que el SQL lleve su origen a logs, trazas y Query Store. Detecto N+1 con un presupuesto de consultas por petición, tanto como alerta como en pruebas de integración. Uso OpenTelemetry para ver cada consulta en la traza de la petición y las estadísticas del motor (Query Store, `pg_stat_statements`) para priorizar por costo total. En desarrollo convierto en excepciones advertencias como las de varias colecciones incluidas.

### Preguntas frecuentes de seguimiento

**1. ¿Para qué sirve `TagWith`?**
Agrega un comentario al SQL que identifica su origen en logs y herramientas del motor.

**2. ¿Cómo detectas un N+1 en producción?**
Contando consultas por petición o viendo cientos de *spans* de SQL en la traza.

**3. ¿Por qué no registrar todos los comandos?**
Por volumen, costo y riesgo de datos sensibles; mejor solo los lentos.

**4. ¿Qué consulta optimizarías primero?**
La de mayor tiempo **total** (duración por cantidad de ejecuciones), no necesariamente la más lenta.

-----

## Práctica

**Ejercicio 1.** Modifica el `MonitorDeConsultas` para que, además, guarde la consulta más lenta (etiqueta y duración) y la imprima al final.

<details>
<summary>Solución</summary>

```csharp
public (string Etiqueta, TimeSpan Duracion) MasLenta { get; private set; } = ("(ninguna)", TimeSpan.Zero);

// dentro de ReaderExecutedAsync, después de agregar la etiqueta:
if (eventData.Duration > MasLenta.Duracion)
{
    MasLenta = (ObtenerEtiqueta(command.CommandText), eventData.Duration);
}

// al final del programa:
Console.WriteLine($"Más lenta: {monitor.MasLenta.Etiqueta} ({monitor.MasLenta.Duracion.TotalMilliseconds:F1} ms)");
```

La duración exacta cambia en cada ejecución, por eso no se puede fijar en una salida de ejemplo; lo útil es la etiqueta, que dice qué código mirar.

</details>

**Ejercicio 2.** Configura un `DbContext` para que, solo en Development, registre todos los comandos con parámetros en la consola, y en cualquier entorno convierta en excepción la advertencia de varias colecciones incluidas.

<details>
<summary>Solución</summary>

```csharp
builder.Services.AddDbContext<AppDbContext>(o =>
{
    o.UseNpgsql(builder.Configuration.GetConnectionString("App"));
    o.ConfigureWarnings(w => w.Throw(RelationalEventId.MultipleCollectionIncludeWarning));

    if (builder.Environment.IsDevelopment())
    {
        o.EnableSensitiveDataLogging();
        o.LogTo(Console.WriteLine, [DbLoggerCategory.Database.Command.Name], LogLevel.Information);
    }
});
```

`LogTo` con la categoría de comandos escribe solo el SQL ejecutado (con duración y, por `EnableSensitiveDataLogging`, con valores). La advertencia de varias colecciones se convierte en excepción en todos los entornos, así que un `Include` doble sin `AsSplitQuery` falla en la primera prueba que lo ejecute.

</details>

-----

## Siguiente lección

[Paginación y consultas eficientes](04-Paginacion%20y%20consultas%20eficientes.md)
