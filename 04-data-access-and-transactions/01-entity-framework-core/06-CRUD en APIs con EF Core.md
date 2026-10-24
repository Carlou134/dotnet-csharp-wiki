# CRUD en APIs con EF Core

## En una frase

Un CRUD de API con EF Core recibe DTOs (nunca entidades), lee con proyecciones, crea con `Add` + `SaveChangesAsync`, actualiza cargando la entidad rastreada o con `ExecuteUpdateAsync`, borra con `ExecuteDeleteAsync` y responde con el código HTTP que corresponde a cada resultado.

-----

## Antes de empezar

Conviene que ya sepas:

* Change tracker y estados: [DbContext, entidades y configuración](02-DbContext%20entidades%20y%20configuracion.md).
* Relaciones y proyecciones: [Relaciones y carga de datos](05-Relaciones%20y%20carga%20de%20datos.md).
* Verbos y códigos de estado: [Principios REST y códigos de estado](../../03-aspnet-core-apis/01-diseno-de-apis-rest/01-Principios%20REST%20y%20codigos%20de%20estado.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **DTO de entrada:** tipo que define exactamente qué datos acepta un endpoint.
* **Actualización masiva (*bulk update*):** `ExecuteUpdateAsync`/`ExecuteDeleteAsync`, que ejecutan un `UPDATE`/`DELETE` directo sin cargar entidades.
* **Grafo de entidades:** una entidad junto con las entidades a las que apuntan sus navegaciones.

-----

## El problema

Un controlador típico de tutorial:

```csharp
[HttpPost("insertar")]
public async Task<ActionResult> insertarTarea([FromBody] Tarea tarea)
{
    tarea.TareaId = Guid.NewGuid();
    tarea.FechaCreacion = new DateTime(2025, 12, 9);
    await _context.AddAsync(tarea);
    await _context.SaveChangesAsync();
    return Ok("Se insertó correctamente");
}

[HttpDelete("eliminar/{id}")]
public async Task<ActionResult> eliminarTarea([FromRoute] Guid id)
{
    var tareaActual = await _context.Tarea.FindAsync(id);
    if (tareaActual != null)
    {
        _context.Remove(tareaActual);
        await _context.SaveChangesAsync();
        return Ok("Se eliminó correctamente");
    }
    return NotFound("No se encontró la tarea");
}
```

Un cliente envía este JSON al `POST`:

```json
{
  "titulo": "Estudiar EF",
  "puntos": 9999,
  "categoria": { "categoriaId": "5a1f...", "nombre": "Categoría inventada" }
}
```

Lo que pasa:

1. `Tarea` es la entidad, así que el cliente puede fijar **cualquier** propiedad: `puntos = 9999` se guarda tal cual (*overposting*).
2. `categoria` es una navegación con un objeto nuevo. `AddAsync` agrega **todo el grafo**: EF Core inserta una categoría que nadie creó desde la pantalla de categorías.
3. El cliente recibe `200` con un texto. No sabe el `id` de lo que creó ni dónde consultarlo.
4. Todas las tareas tienen la fecha `2025-12-09`.
5. Borrar hace dos idas a la base: un `SELECT` para cargar la tarea y un `DELETE` para borrarla.

Las consecuencias:

* el cliente decide campos que debería decidir el servidor;
* filas fantasma en otras tablas;
* una API que no se puede consumir sin leer su código.

-----

## Cómo funciona

### 1. Rutas por recurso, no por verbo

| Operación | ❌ Ruta con verbo | ✔ Ruta REST | Respuesta exitosa |
| --- | --- | --- | --- |
| Listar | `GET /list` | `GET /api/tareas` | `200` + lista |
| Obtener | — | `GET /api/tareas/{id}` | `200` o `404` |
| Crear | `POST /insertar` | `POST /api/tareas` | `201` + `Location` + recurso |
| Reemplazar | `PUT /actualizar/{id}` | `PUT /api/tareas/{id}` | `204` o `404` |
| Borrar | `DELETE /eliminar/{id}` | `DELETE /api/tareas/{id}` | `204` o `404` |

El verbo HTTP ya dice la acción. La ruta nombra el recurso.

### 2. DTOs de entrada y de salida

```csharp
public sealed record CrearTareaRequest(
    [Required, MaxLength(200)] string Titulo,
    string? Descripcion,
    Prioridad Prioridad,
    Guid CategoriaId);

public sealed record TareaDto(Guid Id, string Titulo, Prioridad Prioridad, string Categoria, bool Completada);
```

* El request contiene **solo** lo que el cliente puede decidir. `TareaId`, `FechaCreacion` y `Puntos` no están: los decide el servidor.
* Sin navegaciones en el request, no hay grafo que insertar por accidente.
* El DTO de salida aplana `Categoria.Nombre` y no tiene ciclos.

### 3. Leer: proyección

```csharp
TareaDto? tarea = await db.Tareas
    .Where(t => t.TareaId == id)
    .Select(t => new TareaDto(t.TareaId, t.Titulo, t.Prioridad, t.Categoria!.Nombre, t.Completada))
    .FirstOrDefaultAsync(ct);
```

`t.Categoria!.Nombre` dentro de la proyección se traduce a un `JOIN`; no necesitas `Include`. El `!` solo calla al compilador: en SQL no hay referencia nula que desreferenciar.

### 4. Elegir el método de búsqueda

| Método | Si no hay filas | Si hay varias | Nota |
| --- | --- | --- | --- |
| `FindAsync(id)` | `null` | — (busca por clave) | Mira primero el change tracker; devuelve la entidad rastreada |
| `FirstOrDefaultAsync(...)` | `null` | Toma la primera | Ordena si hay más de una posible |
| `SingleOrDefaultAsync(...)` | `null` | `InvalidOperationException` | Para "debe haber como mucho una" |
| `SingleAsync(...)` / `FirstAsync(...)` | `InvalidOperationException` | Single: excepción | Para "debe existir sí o sí" |

Un `Single(...)` seguido de `if (x != null)` es código muerto: si no hay fila, `Single` ya lanzó la excepción.

### 5. Crear

![Flujo de inserción con EF Core](../../Images/Insertar%20Entity-Framework.png)

```csharp
var tarea = new Tarea
{
    Titulo = request.Titulo.Trim(),
    Prioridad = request.Prioridad,
    CategoriaId = request.CategoriaId,
    FechaCreacion = DateTime.UtcNow
};
db.Tareas.Add(tarea);
await db.SaveChangesAsync(ct);
return CreatedAtAction(nameof(ObtenerPorId), new { id = tarea.TareaId }, dto);
```

* **`Add`, no `AddAsync`.** `Add` no toca la base: solo marca el estado `Added`. `AddAsync` existe para generadores de valores que consultan la base (como *HiLo* en SQL Server); en cualquier otro caso agrega una asignación de `ValueTask` sin beneficio.
* **El Guid lo genera EF Core** en `Add`: no hace falta `Guid.NewGuid()`.
* **Fechas en UTC.** `DateTime.Now` depende de la zona horaria del servidor; con dos servidores en regiones distintas, las fechas no se pueden comparar.
* **`201 Created`** con la URL del recurso en `Location` y el recurso en el cuerpo.

### 6. Actualizar: tres caminos

![Flujo de actualización con EF Core](../../Images/Actualizar%20entity-framework.png)

```text
A) Cargar y modificar          B) Update(entidad)               C) ExecuteUpdateAsync
SELECT ... WHERE Id = @id      (sin SELECT)                     UPDATE ... SET Completada = 1
tarea.Titulo = ...             db.Update(tareaDelCliente)       WHERE Id = @id
SaveChanges:                   SaveChanges:
UPDATE solo columnas           UPDATE de TODAS las columnas     (sin change tracker)
que cambiaron
```

| | A) Cargar y modificar | B) `Update` | C) `ExecuteUpdateAsync` |
| --- | --- | --- | --- |
| Idas a la base | 2 | 1 | 1 |
| Columnas en el `UPDATE` | Solo las cambiadas | Todas | Las que indiques |
| Reglas de la entidad | Se ejecutan | Se saltan | Se saltan |
| Riesgo | Ninguno especial | Pisar columnas con valores que el cliente no mandó | No pasa por el change tracker ni por `SaveChanges` |
| Cuándo | Por defecto | Casi nunca en una API | Cambios simples o masivos (`completar`, `archivar todo lo viejo`) |

En el camino A no llamas a `Update()`: la entidad ya está rastreada y el change tracker detecta los cambios solo.

### 7. Borrar

![Flujo de borrado con EF Core](../../Images/Eliminar%20Entity-Framework.png)

```csharp
int filas = await db.Tareas.Where(t => t.TareaId == id).ExecuteDeleteAsync(ct);
return filas == 0 ? NotFound() : NoContent();
```

Una sola sentencia. `filas` dice cuántas filas se borraron: 0 significa que no existía.

`Find` + `Remove` + `SaveChanges` sigue siendo correcto cuando necesitas la entidad (reglas antes de borrar, eventos de dominio, borrado en cascada que EF Core resuelve en memoria). `ExecuteDeleteAsync` depende de las FKs de la base para la cascada.

### 8. Qué devuelve `SaveChangesAsync`

Un `int` con la cantidad de entidades escritas en la base. No es un "estado de transacción": si algo falla, lanza `DbUpdateException` y no se guarda nada, porque todo el lote viaja en una transacción.

-----

## Ejemplo completo

Proyecto `dotnet new web` con el paquete `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0. `Program.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddDbContext<TareasContext>(o => o.UseSqlite("Data Source=tareas-api.db"));

var app = builder.Build();

// Solo para el ejemplo: recrea la base. En una aplicación real se usan migraciones.
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<TareasContext>();
    db.Database.EnsureDeleted();
    db.Database.EnsureCreated();
}

app.MapControllers();
app.Run();

[ApiController]
[Route("api/tareas")]
public sealed class TareasController(TareasContext db) : ControllerBase
{
    [HttpGet]
    public async Task<List<TareaDto>> Listar(CancellationToken ct) =>
        await db.Tareas
            .OrderBy(t => t.Titulo)
            .Select(t => new TareaDto(t.TareaId, t.Titulo, t.Prioridad, t.Categoria!.Nombre, t.Completada))
            .ToListAsync(ct);

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<TareaDto>> ObtenerPorId(Guid id, CancellationToken ct)
    {
        TareaDto? tarea = await db.Tareas
            .Where(t => t.TareaId == id)
            .Select(t => new TareaDto(t.TareaId, t.Titulo, t.Prioridad, t.Categoria!.Nombre, t.Completada))
            .FirstOrDefaultAsync(ct);

        if (tarea is null)
        {
            return NotFound();
        }
        return tarea;
    }

    [HttpPost]
    public async Task<ActionResult<TareaDto>> Crear(CrearTareaRequest request, CancellationToken ct)
    {
        Categoria? categoria = await db.Categorias.FindAsync([request.CategoriaId], ct);
        if (categoria is null)
        {
            ModelState.AddModelError(nameof(request.CategoriaId), "La categoría no existe.");
            return ValidationProblem(ModelState);
        }

        var tarea = new Tarea
        {
            Titulo = request.Titulo.Trim(),
            Descripcion = request.Descripcion,
            Prioridad = request.Prioridad,
            CategoriaId = categoria.CategoriaId,
            FechaCreacion = DateTime.UtcNow
        };
        db.Tareas.Add(tarea);
        await db.SaveChangesAsync(ct);

        var dto = new TareaDto(tarea.TareaId, tarea.Titulo, tarea.Prioridad, categoria.Nombre, tarea.Completada);
        return CreatedAtAction(nameof(ObtenerPorId), new { id = tarea.TareaId }, dto);
    }

    [HttpPut("{id:guid}")]
    public async Task<IActionResult> Actualizar(Guid id, ActualizarTareaRequest request, CancellationToken ct)
    {
        Tarea? tarea = await db.Tareas.FindAsync([id], ct);
        if (tarea is null)
        {
            return NotFound();
        }

        if (!await db.Categorias.AnyAsync(c => c.CategoriaId == request.CategoriaId, ct))
        {
            ModelState.AddModelError(nameof(request.CategoriaId), "La categoría no existe.");
            return ValidationProblem(ModelState);
        }

        tarea.Titulo = request.Titulo.Trim();
        tarea.Descripcion = request.Descripcion;
        tarea.Prioridad = request.Prioridad;
        tarea.CategoriaId = request.CategoriaId;
        await db.SaveChangesAsync(ct);   // UPDATE solo de las columnas que cambiaron

        return NoContent();
    }

    // Acción sobre el recurso: un UPDATE directo, sin cargar la tarea
    [HttpPost("{id:guid}/completar")]
    public async Task<IActionResult> Completar(Guid id, CancellationToken ct)
    {
        int filas = await db.Tareas
            .Where(t => t.TareaId == id)
            .ExecuteUpdateAsync(s => s.SetProperty(t => t.Completada, true), ct);

        return filas == 0 ? NotFound() : NoContent();
    }

    [HttpDelete("{id:guid}")]
    public async Task<IActionResult> Eliminar(Guid id, CancellationToken ct)
    {
        int filas = await db.Tareas.Where(t => t.TareaId == id).ExecuteDeleteAsync(ct);
        return filas == 0 ? NotFound() : NoContent();
    }
}

public enum Prioridad { Baja = 1, Media = 2, Alta = 3 }

public sealed record CrearTareaRequest(
    [Required, MaxLength(200)] string Titulo,
    string? Descripcion,
    Prioridad Prioridad,
    Guid CategoriaId);

public sealed record ActualizarTareaRequest(
    [Required, MaxLength(200)] string Titulo,
    string? Descripcion,
    Prioridad Prioridad,
    Guid CategoriaId);

public sealed record TareaDto(Guid Id, string Titulo, Prioridad Prioridad, string Categoria, bool Completada);

public class Categoria
{
    public Guid CategoriaId { get; set; }
    public required string Nombre { get; set; }
    public List<Tarea> Tareas { get; } = [];
}

public class Tarea
{
    public Guid TareaId { get; set; }
    public required string Titulo { get; set; }
    public string? Descripcion { get; set; }
    public Prioridad Prioridad { get; set; }
    public int Puntos { get; set; }
    public bool Completada { get; set; }
    public DateTime FechaCreacion { get; set; }
    public Guid CategoriaId { get; set; }
    public Categoria? Categoria { get; set; }
}

public sealed class TareasContext(DbContextOptions<TareasContext> options) : DbContext(options)
{
    public DbSet<Categoria> Categorias => Set<Categoria>();
    public DbSet<Tarea> Tareas => Set<Tarea>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Categoria>(categoria =>
        {
            categoria.Property(c => c.Nombre).HasMaxLength(150);
            categoria.HasData(
                new Categoria { CategoriaId = Guid.Parse("11111111-1111-1111-1111-111111111111"), Nombre = "Personal" },
                new Categoria { CategoriaId = Guid.Parse("22222222-2222-2222-2222-222222222222"), Nombre = "Trabajo" });
        });
        modelBuilder.Entity<Tarea>().Property(t => t.Titulo).HasMaxLength(200);
    }
}
```

Peticiones (el Guid de la tarea cambia en cada ejecución; aquí se abrevia como `{id}`):

```text
GET    /api/tareas
       → 200 []

POST   /api/tareas  {"titulo":"Pagar la luz","prioridad":3,"categoriaId":"11111111-1111-1111-1111-111111111111"}
       → 201 {"id":"{id}","titulo":"Pagar la luz","prioridad":3,"categoria":"Personal","completada":false}
         Location: http://localhost:{puerto}/api/tareas/{id}

POST   /api/tareas  {"titulo":"Otra","prioridad":1,"categoriaId":"99999999-9999-9999-9999-999999999999"}
       → 400 ProblemDetails con "errors": {"CategoriaId": ["La categoría no existe."]}

PUT    /api/tareas/{id}  {"titulo":"Pagar la luz y el gas","prioridad":2,"categoriaId":"22222222-2222-2222-2222-222222222222"}
       → 204

POST   /api/tareas/{id}/completar
       → 204

GET    /api/tareas/{id}
       → 200 {"id":"{id}","titulo":"Pagar la luz y el gas","prioridad":2,"categoria":"Trabajo","completada":true}

DELETE /api/tareas/{id}
       → 204

DELETE /api/tareas/{id}
       → 404
```

Qué observar:

* Si el cliente agrega `"puntos": 9999` o una `"categoria"` anidada al `POST`, el JSON se ignora: `CrearTareaRequest` no tiene esas propiedades.
* El `201` trae `Location`: el cliente sabe dónde consultar lo que creó.
* La categoría inexistente responde `400` con el formato de `ValidationProblem`, no un `500` por violación de FK.
* `Completar` y `Eliminar` ejecutan una sola sentencia cada uno. El segundo `DELETE` responde `404` porque `ExecuteDeleteAsync` devolvió 0.
* `[Required, MaxLength(200)]` está en el **parámetro** del record, donde MVC lo valida. Un título de 201 caracteres responde `400` antes de llegar a la acción.

-----

## Errores comunes

**1. Recibir la entidad como parámetro de la acción.**
Qué pasa: el cliente fija campos que no debería y puede insertar entidades relacionadas a través de las navegaciones.
Por qué: el *model binding* llena toda propiedad pública que encuentre en el JSON, y `Add` agrega el grafo completo.
Arreglo: DTO de entrada con solo los campos permitidos; construye la entidad en el servidor.

**2. `Single` seguido de un `if (x != null)`.**
Qué pasa: `InvalidOperationException: Sequence contains no elements` cuando no hay fila; el `if` nunca sirve.
Por qué: `Single` lanza; no devuelve `null`.
Arreglo: `SingleOrDefaultAsync`/`FirstOrDefaultAsync` + `if`, o `FindAsync` por clave.

**3. `Update(entidad)` con datos del cliente.**
Qué pasa: columnas que el cliente no envió quedan en su valor por defecto (`null`, `0`).
Por qué: `Update` marca **todas** las propiedades como modificadas.
Arreglo: carga la entidad y asigna solo los campos del request, o `ExecuteUpdateAsync` con las columnas explícitas.

**4. Responder `200 OK` con un texto a todo.**
Qué pasa: el cliente no recibe el `id` ni la ubicación, y no distingue crear de actualizar.
Por qué: el contrato HTTP se pierde en un mensaje para humanos.
Arreglo: `201 Created` con `Location`, `204 No Content`, `404`, `400` con `ValidationProblem`.

**5. Creer que `ExecuteUpdateAsync` respeta el change tracker.**
Qué pasa: una entidad ya cargada en el mismo contexto sigue mostrando el valor viejo; un `SaveChanges` posterior puede pisar el cambio.
Por qué: la sentencia va directo a la base sin tocar entidades en memoria.
Arreglo: no mezcles ambos estilos sobre las mismas filas en un mismo contexto.

**6. Llamar a `SaveChanges` dentro de un bucle.**
Qué pasa: N transacciones y N idas a la base para lo que era una operación.
Por qué: cada `SaveChanges` es una unidad de trabajo completa.
Arreglo: haz todos los `Add`/cambios y llama a `SaveChangesAsync` una vez al final.

-----

## Según la versión de .NET

* **EF Core 1.0:** `Add`, `Update`, `Remove` y `SaveChanges` sobre el change tracker.
* **EF Core 7:** `ExecuteUpdate` y `ExecuteDelete`; además, `SaveChanges` ya no abre una transacción explícita cuando solo hay una sentencia.
* **EF Core 9:** `ExecuteUpdate` acepta asignar propiedades de *complex types*.
* **EF Core 10:** `ExecuteUpdateAsync` acepta una lambda normal (no un árbol de expresión) para los *setters*, así puedes agregar `SetProperty` de forma condicional con un `if`.
* **ASP.NET Core 2.1:** `[ApiController]` responde `400` automático cuando el modelo no es válido.

-----

## Cuándo sí y cuándo no

**Carga y modifica cuando:** la actualización tiene reglas, toca varias propiedades o necesitas concurrencia optimista.

**Usa `ExecuteUpdateAsync`/`ExecuteDeleteAsync` cuando:** el cambio es simple y no depende de lógica de la entidad, o afecta muchas filas.

**No uses `ExecuteDeleteAsync` cuando:** la entidad dispara reglas o eventos al borrarse, o la cascada no está configurada en la base.

**No recibas entidades en endpoints cuando:** nunca. Aunque hoy coincidan con el request, mañana la entidad tendrá un campo que el cliente no debe tocar.

-----

## Resumen en 5 líneas

1. DTOs de entrada y salida: el cliente no decide campos del servidor ni inserta grafos.
2. Lee con proyecciones; elige `Find`, `FirstOrDefault` o `Single` según lo que esperas.
3. Crea con `Add` + `SaveChangesAsync` y responde `201` con `Location`.
4. Actualiza cargando la entidad rastreada, o con `ExecuteUpdateAsync` para cambios simples.
5. Borra con `ExecuteDeleteAsync` y usa el número de filas para decidir entre `204` y `404`.

-----

## Para profundizar

<details>
<summary>PUT frente a PATCH</summary>

`PUT` reemplaza el recurso completo: el request trae todos los campos editables y los que falten deberían quedar en su valor por defecto. `PATCH` cambia solo algunos campos. Un `PATCH` simple puede ser un DTO con propiedades opcionales (`string? Titulo`) donde `null` significa "no tocar"; para distinguir "no tocar" de "poner en null" existe JSON Patch (`application/json-patch+json`), que en .NET 10 tiene soporte basado en `System.Text.Json`. Los endpoints de acción, como `/completar`, son una alternativa legible cuando el cambio tiene significado de negocio.

</details>

<details>
<summary>Concurrencia optimista en el PUT</summary>

Si dos usuarios editan la misma tarea, el último `PUT` pisa al primero sin aviso. Agrega un *token de concurrencia* (`[Timestamp] public byte[] Version` en SQL Server), devuélvelo en el DTO de salida y exígelo en el request. EF Core agrega `WHERE Version = @original` al `UPDATE`; si no coincide, lanza `DbUpdateConcurrencyException` y respondes `409 Conflict`. La [lección 7](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md) lo muestra completo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Recibo DTOs, no entidades, para evitar overposting. Para crear uso `Add` y `SaveChangesAsync` y respondo `201`. Para actualizar cargo la entidad, cambio sus propiedades y guardo: el change tracker detecta los cambios. Para borrar uso `ExecuteDeleteAsync` o `Remove` + `SaveChanges`, y respondo `204` o `404`.

### Respuesta ampliada (semi-senior)

Separo DTOs de entrada y salida; las lecturas son proyecciones sin tracking. En escritura cargo la entidad cuando hay reglas o concurrencia optimista, y uso `ExecuteUpdateAsync`/`ExecuteDeleteAsync` para cambios simples o masivos, sabiendo que no pasan por el change tracker. Evito `Update()` con datos del cliente porque marca todas las columnas. Uso `Add` en lugar de `AddAsync` salvo con HiLo, fechas en UTC, `201` con `Location`, `ValidationProblem` para errores del cliente, y una sola llamada a `SaveChangesAsync` por unidad de trabajo.

### Preguntas frecuentes de seguimiento

**1. ¿Hace falta llamar a `Update()` después de cambiar una entidad cargada?**
No. Está rastreada; `SaveChanges` detecta los cambios.

**2. ¿Por qué no `AddAsync`?**
Porque `Add` no accede a la base. `AddAsync` solo sirve para generadores de valores que consultan la base, como HiLo.

**3. ¿`ExecuteDeleteAsync` borra en cascada?**
Solo si la FK de la base tiene `ON DELETE CASCADE`. No carga ni recorre dependientes en memoria.

**4. ¿Qué devuelve `SaveChangesAsync`?**
La cantidad de entidades escritas. Si falla, lanza excepción y no guarda nada.

-----

## Práctica

**Ejercicio 1.** Agrega `PATCH /api/tareas/{id}/prioridad` que reciba `{ "prioridad": 3 }` y la cambie sin cargar la tarea. Si la tarea no existe, `404`.

<details>
<summary>Solución</summary>

```csharp
public sealed record CambiarPrioridadRequest(Prioridad Prioridad);

[HttpPatch("{id:guid}/prioridad")]
public async Task<IActionResult> CambiarPrioridad(Guid id, CambiarPrioridadRequest request, CancellationToken ct)
{
    if (!Enum.IsDefined(request.Prioridad))
    {
        ModelState.AddModelError(nameof(request.Prioridad), "Prioridad inválida.");
        return ValidationProblem(ModelState);
    }

    int filas = await db.Tareas
        .Where(t => t.TareaId == id)
        .ExecuteUpdateAsync(s => s.SetProperty(t => t.Prioridad, request.Prioridad), ct);

    return filas == 0 ? NotFound() : NoContent();
}
```

`Enum.IsDefined` importa: el JSON `{"prioridad": 42}` se deserializa sin error a un `Prioridad` que no existe.

</details>

**Ejercicio 2.** Un endpoint importa 500 tareas así. ¿Qué cambiarías?

```csharp
foreach (var request in requests)
{
    await db.Tareas.AddAsync(new Tarea { Titulo = request.Titulo, CategoriaId = request.CategoriaId });
    await db.SaveChangesAsync();
}
```

<details>
<summary>Solución</summary>

```csharp
var categoriasValidas = await db.Categorias.Select(c => c.CategoriaId).ToHashSetAsync(ct);
if (requests.Any(r => !categoriasValidas.Contains(r.CategoriaId)))
{
    return ValidationProblem(...);
}

db.Tareas.AddRange(requests.Select(r => new Tarea
{
    Titulo = r.Titulo.Trim(),
    CategoriaId = r.CategoriaId,
    FechaCreacion = DateTime.UtcNow
}));
await db.SaveChangesAsync(ct);
```

* `Add`/`AddRange` en lugar de `AddAsync`.
* Un solo `SaveChangesAsync`: una transacción y sentencias agrupadas en lotes, en vez de 500 transacciones.
* Si una fila falla, no queda la importación a medias.
* Las categorías se validan con una consulta, no con 500.

</details>

-----

## Siguiente lección

[CRUD seguro con MVC y EF Core](07-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md)
