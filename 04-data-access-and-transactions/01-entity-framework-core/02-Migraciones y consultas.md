# Migraciones y consultas

## En una frase

Una migración es código versionado que lleva el esquema de la base de un estado a otro y **se revisa antes de aplicarse**; una consulta eficiente pide solo las columnas y filas que necesita, no rastrea lo que no va a modificar y se traduce completa a SQL.

-----

## Antes de empezar

Conviene que ya sepas:

* `DbContext`, entidades, configuración y change tracker: [DbContext, entidades y configuración](01-DbContext%20entidades%20y%20configuracion.md).
* Minimal APIs: [Endpoints y grupos de rutas](../../03-aspnet-core-apis/02-minimal-apis/01-Endpoints%20y%20grupos%20de%20rutas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Migración:** clase con métodos `Up` y `Down` que describen un cambio de esquema.
* **Snapshot del modelo:** archivo que guarda cómo era el modelo en la última migración, para compararlo con el actual.
* **Code First:** flujo en el que el código del modelo conduce la evolución del esquema.
* **No tracking:** consulta cuyos resultados no entran al change tracker.
* **Proyección:** `Select` que devuelve solo los datos necesarios, normalmente en un DTO.

-----

## El problema

El equipo decide que `Name` debe llamarse `FullName`. Renombra la propiedad y genera una migración:

```text
dotnet ef migrations add RenameContactName
An operation was scaffolded that may result in the loss of data. Please review the migration for accuracy.
```

Nadie lee la advertencia. La migración generada es:

```csharp
migrationBuilder.DropColumn(name: "Name", table: "Contacts");
migrationBuilder.AddColumn<string>(name: "FullName", table: "Contacts", ...);
```

Al aplicarla, **todos los nombres de los contactos se borran**. EF Core no puede saber si quisiste renombrar o si quisiste eliminar una columna y crear otra.

Mientras tanto, un endpoint lista contactos así:

```csharp
var contactos = await db.Contacts.ToListAsync();
var corporativos = contactos.Where(c => EsCorporativo(c.Email)).ToList();
```

Trae **toda** la tabla, con todas las columnas, la rastrea entera y filtra en memoria. Con 10 contactos no se nota; con 2 millones, el servidor se queda sin memoria. Y si alguien "lo optimiza" moviendo el filtro antes de `ToListAsync`, aparece otro error:

```text
InvalidOperationException: The LINQ expression 'DbSet<Contact>().Where(c => EsCorporativo(c.Email))'
could not be translated. Either rewrite the query in a form that can be translated, or switch to client
evaluation explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync'.
```

Las consecuencias:

* migraciones que destruyen datos porque nadie las revisó;
* consultas que mueven tablas enteras a memoria;
* métodos de C# dentro de una consulta que EF Core no sabe traducir a SQL.

-----

## Cómo funciona

### 1. Herramientas

| Paquete o herramienta | Para qué |
| --- | --- |
| Proveedor (`Microsoft.EntityFrameworkCore.Sqlite`) | Ejecutar la aplicación contra el motor |
| `Microsoft.EntityFrameworkCore.Design` | Que las herramientas puedan crear el contexto al generar migraciones |
| `dotnet-ef` (herramienta global o local) | Los comandos `dotnet ef ...` |

Todas con la misma versión principal que EF Core.

### 2. Generar una migración

```text
dotnet ef migrations add InitialCreate
```

```text
modelo actual (tu código) ──┐
                            ├──► diferencia ──► Migrations/2026..._InitialCreate.cs (Up / Down)
snapshot anterior ──────────┘                   Migrations/AppDbContextModelSnapshot.cs (actualizado)
```

**Generar no toca la base.** Solo escribe archivos C# que debes leer y versionar en Git.

### 3. Revisar antes de aplicar

Lee siempre `Up` y `Down`. Busca `DropColumn`, `DropTable`, cambios de tipo y columnas nuevas `NOT NULL` sin valor por defecto en tablas con datos. Si la intención era renombrar, corrige la migración a mano:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(name: "Name", table: "Contacts", newName: "FullName");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(name: "FullName", table: "Contacts", newName: "Name");
}
```

### 4. Aplicar

```text
dotnet ef database update
```

La base guarda las migraciones aplicadas en la tabla `__EFMigrationsHistory`. `database update` aplica solo las pendientes, en orden.

En producción, aplicar desde la máquina del desarrollador no es un proceso. Las opciones habituales:

| Estrategia | Cómo | Ventaja |
| --- | --- | --- |
| Script SQL | `dotnet ef migrations script --idempotent` | Un DBA lo revisa; se aplica con las herramientas del motor |
| Bundle | `dotnet ef migrations bundle` | Un ejecutable que el pipeline de despliegue corre una vez |
| `Migrate()` al arrancar | `db.Database.MigrateAsync()` en `Program.cs` | Simple, pero cada réplica intenta migrar y el arranque depende del esquema |

### 5. Consultas: proyecta, no rastrees lo que no modificas

```csharp
ContactRow[] rows = await db.Contacts
    .AsNoTracking()
    .Where(c => c.Email.EndsWith("@empresa.com"))
    .OrderBy(c => c.Id)
    .Select(c => new ContactRow(c.Id, c.Name, c.Email))
    .ToArrayAsync(cancellationToken);
```

* El `Where` se traduce a SQL: filtra la base, no tu memoria.
* El `Select` hacia un DTO trae solo esas columnas. Cuando proyectas a un tipo que no es entidad, EF Core no rastrea nada, así que `AsNoTracking` es redundante pero deja clara la intención.
* `AsNoTracking` sí importa cuando devuelves entidades completas que no vas a modificar.

Para paginar, ordena siempre antes de `Skip`/`Take`: sin `OrderBy`, el orden de las filas no está garantizado.

### 6. Qué se traduce a SQL y qué no

EF Core traduce operadores de LINQ y muchos métodos conocidos (`StartsWith`, `EndsWith`, `Contains`, operaciones de `DateTime`). **No puede traducir tus métodos de C#.** Desde EF Core 3.0, si una parte del `Where` no se traduce, la consulta lanza excepción en lugar de traer la tabla en silencio.

Para ver el SQL que genera una consulta, sin ejecutarla:

```csharp
Console.WriteLine(query.ToQueryString());
```

### 7. Asincronía y cancelación

`ToListAsync`, `SaveChangesAsync` y compañía liberan el hilo mientras la base trabaja. **No vuelven más rápida una consulta mala**: solo permiten atender otras peticiones mientras tanto. Pasa el `CancellationToken` del endpoint hasta EF Core; si el cliente se desconecta, la consulta se cancela.

-----

## Ejemplo completo

Proyecto `dotnet new web`. Usa las mismas clases `Contact` y `AppDbContext` de la [lección anterior](01-DbContext%20entidades%20y%20configuracion.md) (con su `OnModelCreating`).

```text
dotnet package add Microsoft.EntityFrameworkCore.Sqlite --version 10.0.0
dotnet package add Microsoft.EntityFrameworkCore.Design --version 10.0.0
dotnet tool install --global dotnet-ef --version 10.0.0
```

`appsettings.json`:

```json
{
  "ConnectionStrings": { "Database": "Data Source=contacts.db" }
}
```

`Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
var connection = builder.Configuration.GetConnectionString("Database")
    ?? throw new InvalidOperationException("Falta ConnectionStrings:Database.");

builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlite(connection));

var app = builder.Build();

app.MapGet("/contacts", async (AppDbContext db, CancellationToken ct) =>
{
    ContactRow[] contacts = await db.Contacts
        .AsNoTracking()
        .OrderBy(c => c.Id)
        .Select(c => new ContactRow(c.Id, c.Name, c.Email))
        .ToArrayAsync(ct);

    return TypedResults.Ok(contacts);
});

app.MapPost("/contacts", async (CreateContact request, AppDbContext db, CancellationToken ct) =>
{
    var contact = new Contact { Name = request.Name.Trim(), Email = request.Email.Trim() };
    db.Contacts.Add(contact);
    await db.SaveChangesAsync(ct);

    return TypedResults.Created($"/contacts/{contact.Id}",
        new ContactRow(contact.Id, contact.Name, contact.Email));
});

app.Run();

public sealed record CreateContact(string Name, string Email);
public sealed record ContactRow(int Id, string Name, string Email);

// Contact y AppDbContext: copia las clases de la lección anterior.
```

Genera y aplica la primera migración:

```text
dotnet ef migrations add InitialCreate
dotnet ef database update
```

Fragmento de `Up` generado para SQLite:

```csharp
migrationBuilder.CreateTable(
    name: "Contacts",
    columns: table => new
    {
        Id = table.Column<int>(type: "INTEGER", nullable: false)
            .Annotation("Sqlite:Autoincrement", true),
        Name = table.Column<string>(type: "TEXT", maxLength: 120, nullable: false),
        Email = table.Column<string>(type: "TEXT", maxLength: 254, nullable: false)
    },
    constraints: table => table.PrimaryKey("PK_Contacts", x => x.Id));

migrationBuilder.CreateIndex(
    name: "IX_Contacts_Email",
    table: "Contacts",
    column: "Email",
    unique: true);
```

Peticiones:

```text
GET  /contacts                                              → 200 []
POST /contacts  {"name":"Ana","email":"ana@example.com"}    → 201 {"id":1,"name":"Ana","email":"ana@example.com"}
                                                              Location: /contacts/1
GET  /contacts                                              → 200 [{"id":1,"name":"Ana","email":"ana@example.com"}]
```

Qué observar:

* La configuración de la lección anterior (`HasMaxLength`, índice único) aparece en la migración: el esquema expresa las reglas.
* `database update` creó `contacts.db` y la tabla `__EFMigrationsHistory` con una fila: `InitialCreate`.
* Un segundo `POST` con el mismo email lanza `DbUpdateException` por el índice único. La base protege la regla; la [siguiente lección](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md) muestra cómo responder bien en ese caso.

-----

## Errores comunes

**1. Creer que `migrations add` actualiza la base.**
Qué pasa: el esquema no cambia y la aplicación falla con columnas inexistentes.
Por qué: generar y aplicar son pasos distintos.
Arreglo: revisa la migración y aplícala con `database update`, un script o un bundle.

**2. Aplicar una migración sin leerla.**
Qué pasa: un renombrado se convierte en `DropColumn` + `AddColumn` y se pierden datos.
Por qué: EF Core no distingue "renombrar" de "borrar y crear otra".
Arreglo: lee `Up`/`Down`, presta atención a la advertencia de pérdida de datos y usa `RenameColumn` cuando corresponda.

**3. Traer todo y filtrar en memoria.**
Qué pasa: memoria y tiempo proporcionales al tamaño de la tabla.
Por qué: `ToListAsync()` antes del `Where` ejecuta un `SELECT` sin filtro.
Arreglo: filtra y proyecta **antes** de materializar; reemplaza métodos propios por expresiones traducibles.

**4. Usar tracking en lecturas puras.**
Qué pasa: más memoria y CPU por cada fila leída.
Por qué: el change tracker guarda una copia del estado de cada entidad.
Arreglo: proyecta a DTOs o usa `AsNoTracking`.

**5. Aplicar migraciones al arrancar sin estrategia.**
Qué pasa: varias réplicas intentan migrar a la vez, o un cambio largo bloquea tablas mientras la app arranca.
Por qué: el despliegue de esquema necesita coordinación.
Arreglo: un paso del pipeline (script revisado o bundle) antes de desplegar la nueva versión.

**6. Mezclar versiones principales de EF Core.**
Qué pasa: errores al restaurar, al generar migraciones o en ejecución.
Por qué: proveedor, `Design` y `dotnet-ef` se publican juntos para cada versión principal.
Arreglo: alinea todos en la misma versión (por ejemplo, 10.0.x).

-----

## Según la versión de .NET

* **EF6:** migraciones propias de la línea clásica (`Add-Migration`, `Update-Database` en la consola del administrador de paquetes).
* **EF Core 1.0+:** migraciones con `dotnet ef` y `Microsoft.EntityFrameworkCore.Design`.
* **EF Core 3.0:** las consultas no traducibles lanzan excepción; se acaba la evaluación silenciosa en memoria.
* **EF Core 5:** `ToQueryString()` para ver el SQL generado.
* **EF Core 6:** *migration bundles*.
* **EF Core 9:** `Migrate()` toma un bloqueo para que dos instancias no apliquen migraciones a la vez.
* **EF Core 10:** acompaña a .NET 10; `dotnet package add` requiere el SDK 10, y `dotnet ef` conserva su espacio de comandos.

-----

## Cuándo sí y cuándo no

**Usa migraciones cuando:** el esquema evoluciona con el código, el equipo es dueño de la base y puedes revisar cada cambio.

**No las uses tal cual cuando:** la base es compartida con otros sistemas o la administra otro equipo. Ahí conviene generar scripts para revisión, o trabajar *Database First* con `dotnet ef dbcontext scaffold`.

**Usa `AsNoTracking` o proyecciones cuando:** solo lees. **No las uses cuando:** vas a modificar y guardar esas mismas entidades.

-----

## Resumen en 5 líneas

1. Una migración se genera comparando el modelo actual con el snapshot; generar no cambia la base.
2. Toda migración se lee antes de aplicarse: EF Core no distingue un renombrado de un borrado.
3. En producción, las migraciones se aplican con un paso controlado (script o bundle).
4. Filtra y proyecta antes de materializar; tus métodos de C# no se traducen a SQL.
5. Async libera hilos y la cancelación corta trabajo inútil, pero ninguna arregla una consulta mala.

-----

## Para profundizar

<details>
<summary>¿Por qué revisar también `Down`?</summary>

`Down` revierte la migración, pero no todo se revierte sin pérdida: si `Up` borra una columna, `Down` la vuelve a crear vacía. Un rollback real puede necesitar un respaldo o una migración compensatoria. Trata `Down` como una herramienta de desarrollo, no como un plan de recuperación en producción.

</details>

<details>
<summary>Cambios de esquema sin cortar el servicio</summary>

Para renombrar una columna en un sistema que no puede detenerse, se usa el patrón *expand/contract*: (1) agregar la columna nueva; (2) desplegar código que escribe en ambas; (3) copiar datos; (4) desplegar código que solo lee la nueva; (5) borrar la vieja en una migración posterior. Cada paso es una migración pequeña y reversible.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las migraciones versionan cambios del modelo al esquema. Primero se generan con `dotnet ef migrations add`, se revisan y después se aplican con `database update`. En las consultas uso `AsNoTracking` o proyecciones cuando solo leo, y filtro antes de traer los datos.

### Respuesta ampliada (semi-senior)

EF Core compara el modelo con el snapshot y genera `Up`/`Down`. Reviso cada migración, sobre todo renombrados y cambios de tipo, porque EF Core no distingue renombrar de borrar. En producción aplico con scripts idempotentes o bundles desde el pipeline, no al arrancar cada réplica, y para cambios grandes uso expand/contract. En las consultas proyecto a DTOs, pagino con orden estable, verifico el SQL con `ToQueryString` y propago la cancelación.

### Preguntas frecuentes de seguimiento

**1. ¿`database update` crea la base?**
Si no existe y el proveedor lo permite, sí; su función principal es aplicar migraciones pendientes.

**2. ¿`AsNoTracking` sirve para editar?**
No. Esas entidades no están rastreadas: cambiarlas no genera `UPDATE`.

**3. ¿Por qué falló mi `Where` con un método propio?**
Porque EF Core no puede traducir un método arbitrario de C# a SQL.

**4. ¿Dónde se guarda qué migraciones se aplicaron?**
En la tabla `__EFMigrationsHistory` de la propia base.

-----

## Práctica

**Ejercicio 1.** Agrega una propiedad `Phone` opcional a `Contact`. ¿Qué tipo usas, qué esperas ver en la migración y qué revisas antes de aplicarla?

<details>
<summary>Solución</summary>

`public string? Phone { get; set; }`, con `HasMaxLength(30)` en la configuración. La migración debe tener un `AddColumn<string>` con `nullable: true` y `maxLength: 30`, y ningún `Drop`. Como es nullable, las filas existentes quedan con `NULL` y la migración no falla. Si fuera obligatoria (`string`), la columna sería `NOT NULL` y necesitaría un valor por defecto para las filas existentes.

</details>

**Ejercicio 2.** Reescribe el filtro del problema para que se ejecute en la base. `EsCorporativo` devolvía `true` si el email terminaba en `@empresa.com`.

<details>
<summary>Solución</summary>

```csharp
ContactRow[] corporativos = await db.Contacts
    .Where(c => c.Email.EndsWith("@empresa.com"))
    .OrderBy(c => c.Id)
    .Select(c => new ContactRow(c.Id, c.Name, c.Email))
    .ToArrayAsync(cancellationToken);
```

`EndsWith` con una constante sí se traduce (a `LIKE` o a una comparación equivalente, según el proveedor). La base filtra, solo viajan tres columnas y nada entra al change tracker.

</details>

-----

## Siguiente lección

[CRUD seguro con MVC y EF Core](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md)
