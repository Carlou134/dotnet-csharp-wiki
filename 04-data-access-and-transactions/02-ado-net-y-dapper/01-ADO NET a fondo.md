# ADO.NET a fondo

## En una frase

Con ADO.NET abres una conexión (que en realidad sale de un *pool*), ejecutas un comando con parámetros tipados y lees filas con un `DbDataReader`; todo lo que abras lo cierras con `await using`, porque una conexión que no vuelve al pool tarde o temprano tumba la aplicación.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es ADO.NET y por qué existen los parámetros: [ADO.NET, ORMs y EF Core](../01-entity-framework-core/01-ADO%20NET%20ORMs%20y%20EF%20Core.md).
* `IDisposable`, `using` y `await using`: [Asincronía y archivos](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).
* SQL básico: `SELECT`, `INSERT`, `UPDATE`, `WHERE`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Pool de conexiones:** conjunto de conexiones físicas abiertas que se reutilizan; `Open` toma una y `Dispose` la devuelve.
* **`DbCommand`:** objeto que contiene el texto SQL, los parámetros y la transacción con la que se ejecuta.
* **`DbDataReader`:** cursor de solo avance y solo lectura sobre las filas que devuelve un comando.
* **Ordinal:** posición numérica de una columna dentro del resultado (0, 1, 2...).
* **`DBNull.Value`:** valor que representa `NULL` de SQL en ADO.NET; no es lo mismo que `null` de C#.
* **Conjunto de resultados (*result set*):** cada bloque de filas que devuelve un comando; un comando puede devolver varios.

-----

## El problema

Un repositorio de tareas escrito con prisa:

```csharp
public List<Tarea> Pendientes()
{
    var conexion = new SqlConnection(_cadena);
    conexion.Open();

    var cmd = new SqlCommand("SELECT * FROM Tareas WHERE Completada = 0", conexion);
    var reader = cmd.ExecuteReader();

    var lista = new List<Tarea>();
    while (reader.Read())
    {
        lista.Add(new Tarea
        {
            Id = reader.GetInt32(0),
            Titulo = reader.GetString(1),
            Descripcion = reader.GetString(2)
        });
    }
    return lista;
}

public void Crear(string titulo, string? descripcion)
{
    using var conexion = new SqlConnection(_cadena);
    conexion.Open();
    var cmd = new SqlCommand("INSERT INTO Tareas (Titulo, Descripcion) VALUES (@t, @d)", conexion);
    cmd.Parameters.AddWithValue("@t", titulo);
    cmd.Parameters.AddWithValue("@d", descripcion);
    cmd.ExecuteNonQuery();
}
```

Lo que pasa en producción:

1. `Pendientes` nunca cierra la conexión. Cada llamada se queda con una del pool. A la llamada 101, SQL Server espera 15 segundos y lanza `InvalidOperationException: Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool... max pool size was reached`. La API entera deja de responder, no solo este endpoint.
2. Una tarea sin descripción hace fallar la lectura: `GetString(2)` sobre `NULL` lanza `SqlNullValueException: Data is Null`.
3. `Crear` con `descripcion = null` falla: `AddWithValue("@d", null)` no envía `NULL`, sino un parámetro sin valor, y SQL Server responde `The parameterized query ... expects the parameter '@d', which was not supplied`.
4. `SELECT *` con ordinales fijos: alguien agrega una columna en medio de la tabla y `GetString(1)` lee otra cosa.

Las consecuencias:

* una fuga de conexiones que se ve como "la base está caída";
* errores con datos opcionales que en desarrollo nunca aparecieron;
* lecturas que se rompen con cambios de esquema inocentes.

-----

## Cómo funciona

### 1. El ciclo de vida de una conexión

```text
new SqlConnection(cadena)   → objeto en memoria, sin red
await OpenAsync()           → toma una conexión física del pool (o crea una si no hay libres)
... comandos ...
await DisposeAsync()        → la devuelve al pool (sigue abierta físicamente, lista para otro)
```

```text
Pool para "Server=db;Database=Tareas;..."   (uno por cadena de conexión exacta)
┌─────────┬─────────┬─────────┬─────────┐
│ libre   │ en uso  │ en uso  │ libre   │   ... hasta Max Pool Size (100 por defecto en SQL Server)
└─────────┴─────────┴─────────┴─────────┘
   ▲ Open toma una libre        Dispose la devuelve ▼
```

Consecuencias prácticas:

* **Abre tarde, cierra pronto.** Abrir es barato gracias al pool; retener una conexión es caro porque nadie más puede usarla.
* **Siempre `await using`.** Si salta una excepción entre `Open` y el cierre manual, la conexión se pierde.
* Dos cadenas que difieren en un espacio o en el orden de las claves crean **dos pools distintos**.

### 2. Comandos: tres formas de ejecutar

| Método | Devuelve | Para |
| --- | --- | --- |
| `ExecuteNonQueryAsync` | `int`: filas afectadas | `INSERT`, `UPDATE`, `DELETE`, DDL |
| `ExecuteScalarAsync` | `object?`: primera columna de la primera fila | `COUNT(*)`, `INSERT ... RETURNING Id` |
| `ExecuteReaderAsync` | `DbDataReader` | `SELECT` con varias filas o columnas |

`ExecuteScalarAsync` devuelve `object?`: hay que convertir. En SQLite, un `COUNT(*)` llega como `long`; en SQL Server, como `int`.

### 3. Parámetros tipados

```csharp
// ❌ El proveedor adivina el tipo a partir del valor
cmd.Parameters.AddWithValue("@titulo", titulo);

// ✔ Tipo y tamaño explícitos
cmd.Parameters.Add("@titulo", SqlDbType.NVarChar, 200).Value = titulo;
cmd.Parameters.Add("@descripcion", SqlDbType.NVarChar, 1000).Value = (object?)descripcion ?? DBNull.Value;
```

Por qué importa en SQL Server:

* Un `string` de C# se envía como `nvarchar`. Si la columna es `varchar`, el motor convierte **cada fila** de la columna para comparar y deja de usar el índice.
* `AddWithValue` usa la longitud del valor: `"Ana"` va como `nvarchar(3)` y `"Luis"` como `nvarchar(4)`. Cada longitud genera un plan de ejecución distinto en la caché.
* `null` no es `NULL`: hay que pasar `DBNull.Value`.

### 4. Leer con el reader

```csharp
await using var reader = await cmd.ExecuteReaderAsync(ct);

int colId = reader.GetOrdinal("Id");                // busca el ordinal una vez, por nombre
int colTitulo = reader.GetOrdinal("Titulo");
int colDescripcion = reader.GetOrdinal("Descripcion");

while (await reader.ReadAsync(ct))
{
    var tarea = new TareaFila(
        reader.GetInt32(colId),
        reader.GetString(colTitulo),
        reader.IsDBNull(colDescripcion) ? null : reader.GetString(colDescripcion));
}
```

* `GetOrdinal` por nombre, fuera del bucle: resistente a cambios de orden y sin buscar el nombre en cada fila.
* `IsDBNull` antes de leer cualquier columna que admita `NULL`.
* `SELECT` con las columnas explícitas, nunca `SELECT *`.
* El reader mantiene la conexión **ocupada** hasta que se cierra: no ejecutes otro comando en la misma conexión mientras lees (salvo MARS en SQL Server).

### 5. Varios resultados en un comando

```csharp
cmd.CommandText = "SELECT COUNT(*) FROM Tareas; SELECT COUNT(*) FROM Tareas WHERE Completada = 1;";
await using var reader = await cmd.ExecuteReaderAsync(ct);

await reader.ReadAsync(ct);
long total = reader.GetInt64(0);

await reader.NextResultAsync(ct);   // pasa al segundo SELECT
await reader.ReadAsync(ct);
long completadas = reader.GetInt64(0);
```

Una sola ida y vuelta a la base para dos consultas. Es la base de `QueryMultiple` en Dapper.

### 6. Reutilizar un comando

Para insertar muchas filas, crea los parámetros una vez y cambia solo sus valores:

```csharp
cmd.CommandText = "INSERT INTO Tareas (Titulo) VALUES ($titulo)";
var pTitulo = cmd.Parameters.Add("$titulo", SqliteType.Text);

foreach (var titulo in titulos)
{
    pTitulo.Value = titulo;
    await cmd.ExecuteNonQueryAsync(ct);
}
```

Muchas filas en un bucle merecen además una transacción ([lección 4](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md)): sin ella, cada `INSERT` es su propia transacción con su propia escritura a disco.

### 7. Asincronía y cancelación

Usa las versiones `Async` y pasa el `CancellationToken`. Liberan el hilo mientras la base trabaja y permiten cortar una consulta si el cliente se desconecta. Un aviso honesto: algunos proveedores (Microsoft.Data.Sqlite entre ellos) implementan los métodos `Async` de forma síncrona por debajo. Con SQLite no ganas concurrencia; con SQL Server o PostgreSQL, sí.

-----

## Ejemplo completo

Consola con el proveedor ADO.NET de SQLite. `AdoNet.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Data.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using Microsoft.Data.Sqlite;

await using var conexion = new SqliteConnection("Data Source=adonet.db");
await conexion.OpenAsync();

// 1. DDL con ExecuteNonQuery
await using (var cmd = conexion.CreateCommand())
{
    cmd.CommandText = """
        DROP TABLE IF EXISTS Tareas;
        CREATE TABLE Tareas (
            Id          INTEGER PRIMARY KEY,
            Titulo      TEXT    NOT NULL,
            Descripcion TEXT    NULL,
            Completada  INTEGER NOT NULL DEFAULT 0
        );
        """;
    await cmd.ExecuteNonQueryAsync();
}

// 2. Un comando reutilizado con parámetros tipados; null → DBNull.Value
(string Titulo, string? Descripcion, bool Completada)[] datos =
[
    ("Estudiar ADO.NET", null, false),
    ("Pagar la luz", "Vence el 10", true),
    ("Revisar PR", "Antes del viernes", false)
];

int insertadas = 0;
await using (var cmd = conexion.CreateCommand())
{
    cmd.CommandText = "INSERT INTO Tareas (Titulo, Descripcion, Completada) VALUES ($titulo, $descripcion, $completada)";
    var pTitulo = cmd.Parameters.Add("$titulo", SqliteType.Text);
    var pDescripcion = cmd.Parameters.Add("$descripcion", SqliteType.Text);
    var pCompletada = cmd.Parameters.Add("$completada", SqliteType.Integer);

    foreach (var (titulo, descripcion, completada) in datos)
    {
        pTitulo.Value = titulo;
        pDescripcion.Value = (object?)descripcion ?? DBNull.Value;
        pCompletada.Value = completada;
        insertadas += await cmd.ExecuteNonQueryAsync();
    }
}
Console.WriteLine($"Filas insertadas: {insertadas}");

// 3. ExecuteScalar: object? que hay que convertir
await using (var cmd = conexion.CreateCommand())
{
    cmd.CommandText = "SELECT COUNT(*) FROM Tareas WHERE Completada = $completada";
    cmd.Parameters.AddWithValue("$completada", false);
    long pendientes = (long)(await cmd.ExecuteScalarAsync())!;
    Console.WriteLine($"Pendientes: {pendientes}");
}

// 4. Reader: ordinales por nombre e IsDBNull
await using (var cmd = conexion.CreateCommand())
{
    cmd.CommandText = "SELECT Id, Titulo, Descripcion FROM Tareas WHERE Completada = 0 ORDER BY Id";
    await using var reader = await cmd.ExecuteReaderAsync();

    int colTitulo = reader.GetOrdinal("Titulo");
    int colDescripcion = reader.GetOrdinal("Descripcion");

    while (await reader.ReadAsync())
    {
        string titulo = reader.GetString(colTitulo);
        string descripcion = reader.IsDBNull(colDescripcion) ? "(sin descripción)" : reader.GetString(colDescripcion);
        Console.WriteLine($"- {titulo}: {descripcion}");
    }
}

// 5. Dos resultados en una sola ida a la base
await using (var cmd = conexion.CreateCommand())
{
    cmd.CommandText = "SELECT COUNT(*) FROM Tareas; SELECT COUNT(*) FROM Tareas WHERE Completada = 1;";
    await using var reader = await cmd.ExecuteReaderAsync();

    await reader.ReadAsync();
    Console.WriteLine($"Resultado 1: {reader.GetInt64(0)} tareas");

    await reader.NextResultAsync();
    await reader.ReadAsync();
    Console.WriteLine($"Resultado 2: {reader.GetInt64(0)} completadas");
}

// 6. ExecuteNonQuery devuelve filas afectadas
await using (var cmd = conexion.CreateCommand())
{
    cmd.CommandText = "UPDATE Tareas SET Completada = 1 WHERE Completada = 0";
    Console.WriteLine($"Tareas completadas ahora: {await cmd.ExecuteNonQueryAsync()}");
}
```

Salida:

```text
Filas insertadas: 3
Pendientes: 2
- Estudiar ADO.NET: (sin descripción)
- Revisar PR: Antes del viernes
Resultado 1: 3 tareas
Resultado 2: 1 completadas
Tareas completadas ahora: 2
```

Qué observar:

* La descripción `null` llegó a la base como `NULL` gracias a `DBNull.Value`, y `IsDBNull` evitó la excepción al leerla.
* `ExecuteScalarAsync` devolvió un `long`: así representa SQLite los enteros. En SQL Server, `COUNT(*)` llega como `int`; el *cast* depende del proveedor.
* Los ordinales salieron de `GetOrdinal`, no de números fijos.
* El quinto bloque hizo dos consultas con un solo comando gracias a `NextResultAsync`.
* Cada comando y cada reader está en un `await using`: nada queda abierto aunque falle algo en el medio.

-----

## Errores comunes

**1. No cerrar la conexión o el reader.**
Qué pasa: `InvalidOperationException: Timeout expired... max pool size was reached` cuando se agota el pool.
Por qué: la conexión solo vuelve al pool con `Dispose`/`Close`.
Arreglo: `await using` en conexión, comando y reader. Abre tarde, cierra pronto.

**2. Pasar `null` como valor de un parámetro.**
Qué pasa: SQL Server dice que el parámetro "was not supplied"; Microsoft.Data.Sqlite lanza `InvalidOperationException` por un parámetro sin valor.
Por qué: ADO.NET representa `NULL` con `DBNull.Value`; `null` significa "sin valor asignado".
Arreglo: `(object?)valor ?? DBNull.Value`.

**3. Leer un `NULL` sin comprobarlo.**
Qué pasa: `SqlNullValueException` (SQL Server) o `InvalidCastException`, según el proveedor.
Por qué: `GetString`, `GetInt32` y compañía no saben devolver "nada".
Arreglo: `IsDBNull(ordinal)` antes de leer, o `GetFieldValue<int?>` donde el proveedor lo soporte.

**4. `AddWithValue` en SQL Server con columnas `varchar`.**
Qué pasa: consultas lentas que hacen *scan* completo aunque haya índice, y caché de planes llena de variantes.
Por qué: el tipo y la longitud se infieren del valor de C# (`nvarchar(n)`), no de la columna.
Arreglo: `Parameters.Add(nombre, SqlDbType.VarChar, 200).Value = ...`.

**5. Ejecutar otro comando mientras un reader está abierto.**
Qué pasa: `InvalidOperationException: There is already an open DataReader associated with this Connection`.
Por qué: la conexión está ocupada entregando filas.
Arreglo: lee todo y cierra el reader antes del siguiente comando, o usa otra conexión.

**6. Ordinales fijos con `SELECT *`.**
Qué pasa: lecturas de la columna equivocada o `InvalidCastException` tras un cambio de esquema.
Por qué: el orden de las columnas de `*` depende de la tabla, no de tu código.
Arreglo: columnas explícitas en el `SELECT` y `GetOrdinal` por nombre.

-----

## Según la versión de .NET

* **.NET Framework 1.0 (2002):** ADO.NET con `SqlConnection`, `SqlCommand`, `SqlDataReader`, `DataSet`.
* **.NET Framework 4.5:** métodos `Async` (`OpenAsync`, `ExecuteReaderAsync`, `ReadAsync`).
* **.NET Core 3.0 / C# 8:** `IAsyncDisposable` y `await using` en conexiones, comandos y readers.
* **2019:** `Microsoft.Data.SqlClient` reemplaza a `System.Data.SqlClient` como proveedor de SQL Server con soporte activo.
* **.NET 6:** Microsoft.Data.Sqlite agrega pool de conexiones.
* **.NET 7:** `DbDataSource`, una fábrica de conexiones configurada una vez (muy usada con Npgsql).
* **C# 11:** literales de cadena sin formato (`"""`), cómodos para SQL de varias líneas.

-----

## Cuándo sí y cuándo no

**Usa ADO.NET directo cuando:** necesitas el máximo control (streaming de millones de filas, *bulk copy* con `SqlBulkCopy`, tipos específicos del proveedor) o escribes una biblioteca de infraestructura.

**Usa Dapper encima cuando:** quieres SQL propio sin escribir a mano el mapeo de cada columna. Es lo mismo que hace este ejemplo, en una línea ([lección 2](02-Dapper%20consultas%20y%20comandos.md)).

**No uses ADO.NET a mano cuando:** solo repetirías `GetOrdinal`, `IsDBNull` y `new` para cada consulta del proyecto.

-----

## Resumen en 5 líneas

1. `Open` toma una conexión del pool y `Dispose` la devuelve: siempre `await using`.
2. `ExecuteNonQuery` (filas afectadas), `ExecuteScalar` (un valor) y `ExecuteReader` (filas).
3. Parámetros con tipo explícito; `NULL` se envía como `DBNull.Value`.
4. Lee con `GetOrdinal` por nombre e `IsDBNull` antes de cada columna opcional.
5. `NextResult` permite varias consultas en una sola ida a la base.

-----

## Para profundizar

<details>
<summary><code>DataSet</code> y <code>DataTable</code>: el ADO.NET desconectado</summary>

Además del reader, ADO.NET tiene un modelo "desconectado": un `DataAdapter` llena un `DataSet` (tablas en memoria con filas y columnas sin tipo fuerte), lo editas y luego el adapter genera los `INSERT`/`UPDATE`/`DELETE`. Fue el estándar en WinForms y WebForms. Hoy casi no se usa en código nuevo: un ORM o Dapper con clases tipadas cubren lo mismo con más seguridad de tipos. Lo vas a encontrar en sistemas heredados.

</details>

<details>
<summary><code>CommandBehavior</code> y streaming de columnas grandes</summary>

`ExecuteReaderAsync(CommandBehavior.SequentialAccess)` lee las columnas en orden y sin cargar la fila entera en memoria. Combinado con `reader.GetStream(ordinal)`, permite copiar un `varbinary(max)` de varios cientos de MB a un archivo sin tenerlo completo en RAM. `CommandBehavior.CloseConnection` cierra la conexión al cerrar el reader, útil cuando devuelves el reader a quien te llamó.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Con ADO.NET abro una conexión, creo un comando con parámetros y lo ejecuto: `ExecuteNonQuery` para escribir, `ExecuteScalar` para un valor y `ExecuteReader` para leer filas. Uso `using` para cerrar todo y `DBNull.Value` para los `NULL`.

### Respuesta ampliada (semi-senior)

`Open` toma una conexión del pool (uno por cadena de conexión) y `Dispose` la devuelve; por eso abro tarde, cierro pronto y uso `await using` en conexión, comando y reader. Los parámetros van con tipo y tamaño explícitos: `AddWithValue` en SQL Server infiere `nvarchar(n)`, rompe índices sobre `varchar` y llena la caché de planes. Leo con `GetOrdinal` fuera del bucle e `IsDBNull` en columnas opcionales, nunca `SELECT *`. Uso `NextResult` para varias consultas en una ida y paso el `CancellationToken` a los métodos `Async`.

### Preguntas frecuentes de seguimiento

**1. ¿Abrir una conexión es caro?**
La primera vez sí (red, autenticación). Después, `Open` toma una del pool y es barato; lo caro es retenerla.

**2. ¿Qué diferencia hay entre `null` y `DBNull.Value`?**
`null` es "sin valor asignado"; `DBNull.Value` representa el `NULL` de SQL.

**3. ¿Por qué no `AddWithValue`?**
Porque infiere tipo y longitud del valor de C#, no de la columna: conversiones implícitas y planes duplicados.

**4. ¿Puedo ejecutar dos comandos a la vez en una conexión?**
No mientras un reader esté abierto, salvo MARS en SQL Server. Usa conexiones distintas.

-----

## Práctica

**Ejercicio 1.** Reescribe `Pendientes()` del problema para que no pierda conexiones, tolere descripciones nulas y no dependa del orden de las columnas.

<details>
<summary>Solución</summary>

```csharp
public async Task<List<TareaFila>> PendientesAsync(CancellationToken ct)
{
    await using var conexion = new SqlConnection(_cadena);
    await conexion.OpenAsync(ct);

    await using var cmd = new SqlCommand(
        "SELECT Id, Titulo, Descripcion FROM Tareas WHERE Completada = 0 ORDER BY Id", conexion);
    await using var reader = await cmd.ExecuteReaderAsync(ct);

    int colId = reader.GetOrdinal("Id");
    int colTitulo = reader.GetOrdinal("Titulo");
    int colDescripcion = reader.GetOrdinal("Descripcion");

    var lista = new List<TareaFila>();
    while (await reader.ReadAsync(ct))
    {
        lista.Add(new TareaFila(
            reader.GetInt32(colId),
            reader.GetString(colTitulo),
            reader.IsDBNull(colDescripcion) ? null : reader.GetString(colDescripcion)));
    }
    return lista;
}

public sealed record TareaFila(int Id, string Titulo, string? Descripcion);
```

</details>

**Ejercicio 2.** Escribe `CrearAsync(string titulo, string? descripcion)` para SQL Server que devuelva el `Id` generado en la misma sentencia.

<details>
<summary>Solución</summary>

```csharp
public async Task<int> CrearAsync(string titulo, string? descripcion, CancellationToken ct)
{
    await using var conexion = new SqlConnection(_cadena);
    await conexion.OpenAsync(ct);

    await using var cmd = new SqlCommand(
        "INSERT INTO Tareas (Titulo, Descripcion) OUTPUT INSERTED.Id VALUES (@titulo, @descripcion)", conexion);
    cmd.Parameters.Add("@titulo", SqlDbType.NVarChar, 200).Value = titulo;
    cmd.Parameters.Add("@descripcion", SqlDbType.NVarChar, 1000).Value = (object?)descripcion ?? DBNull.Value;

    return (int)(await cmd.ExecuteScalarAsync(ct))!;
}
```

`OUTPUT INSERTED.Id` devuelve el valor generado en la misma ida a la base. En SQLite y PostgreSQL el equivalente es `INSERT ... RETURNING Id`.

</details>

-----

## Siguiente lección

[Dapper: consultas y comandos](02-Dapper%20consultas%20y%20comandos.md)
