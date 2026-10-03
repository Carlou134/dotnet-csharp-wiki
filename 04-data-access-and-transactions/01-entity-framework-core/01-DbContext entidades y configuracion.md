# DbContext, entidades y configuración

## En una frase
`DbContext` representa una sesión corta con una base de datos: construye el modelo, traduce consultas, rastrea cambios y confirma una unidad de trabajo.

-----

## Antes de empezar
Conviene que ya sepas: [Host, configuración y pipeline HTTP](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)): **proveedor:** adaptador para un motor; **DbSet:** acceso tipado; **unidad de trabajo:** grupo de cambios confirmados juntos.

-----

## El problema
Crear una conexión o `DbContext` global y usarlo desde todos los controladores acumula entidades rastreadas, mezcla operaciones concurrentes y vuelve impredecible el guardado. `DbContext` **no es thread-safe**.

-----

## Cómo funciona
### 1. Paquetes por proveedor
EF Core no se conecta «a cualquier base» sin configuración. Cada motor necesita un proveedor compatible. SQLite es portable; LocalDB y autenticación integrada son opciones específicas de Windows/SQL Server.

### 2. Entidad
```csharp
public sealed class Contact
{
    public int Id { get; private set; }
    public required string Name { get; set; }
    public required string Email { get; set; }
}
```

Una entidad no necesita heredar de EF Core. Las reglas simples pueden expresarse por convención, atributos o Fluent API; las reglas de negocio siguen perteneciendo al dominio.

### 3. Contexto
```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    public DbSet<Contact> Contacts => Set<Contact>();
}
```

### 4. Registro scoped
`AddDbContext` registra el contexto como *scoped* por defecto. En una aplicación web, cada solicitud obtiene normalmente su propia instancia.

### 5. Configuración
```json
{
  "ConnectionStrings": {
    "Database": "Data Source=contacts.db"
  }
}
```

El archivo puede contener configuración local no secreta. Credenciales reales deben provenir de secretos de desarrollo, variables o un gestor de secretos.

### 6. Cambios
`Add`, `Update` y `Remove` cambian el estado rastreado. `SaveChangesAsync` genera comandos y los envía. Agregar una entidad al contexto no significa que ya exista en la base.

-----

## Ejemplo completo
`Contacts.Api.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:
```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
string connection = builder.Configuration.GetConnectionString("Database")
    ?? throw new InvalidOperationException("Falta ConnectionStrings:Database.");

builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlite(connection));

var app = builder.Build();
app.MapGet("/contacts", async (AppDbContext db, CancellationToken ct) =>
    await db.Contacts.AsNoTracking().OrderBy(x => x.Id).ToListAsync(ct));
app.Run();

public sealed class Contact
{
    public int Id { get; private set; }
    public required string Name { get; set; }
    public required string Email { get; set; }
}

public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    public DbSet<Contact> Contacts => Set<Contact>();
}
```

Con una base recién migrada y sin filas, `GET /contacts` devuelve:
```json
[]
```

-----

## Errores comunes
**1. Registrar el contexto como singleton.** Qué pasa: varias solicitudes comparten estado y aparecen errores de concurrencia. Por qué: `DbContext` no es thread-safe. Arreglo: usa el lifetime scoped habitual.

**2. Guardar contraseñas en Git.** Qué pasa: una credencial queda expuesta incluso después de borrar la línea. Por qué: el historial conserva el secreto. Arreglo: rota la credencial y usa un proveedor seguro.

**3. Creer que LocalDB es SQLite.** Qué pasa: se asumen portabilidad y formato incorrectos. Por qué: LocalDB es una variante de SQL Server para desarrollo en Windows. Arreglo: nombra y configura el proveedor real.

-----

## Según la versión de .NET
- **EF6:** pertenece a la generación clásica y no es EF Core.
- **EF Core 1+:** rediseñó el ORM para .NET moderno.
- **EF Core 10:** corresponde a la generación de .NET 10; alinea las versiones principales de los paquetes del proveedor y herramientas.

-----

## Cuándo sí y cuándo no
**Usa EF Core cuando:** el modelo relacional y LINQ reducen trabajo sin ocultar requisitos críticos. **No lo uses a ciegas cuando:** necesitas SQL altamente específico o control fino; puedes combinarlo con SQL explícito y medición.

-----

## Resumen en 5 líneas
1. `DbContext` es una sesión corta y no es thread-safe.
2. Cada motor necesita un proveedor de EF Core.
3. `DbSet<T>` permite consultar y registrar cambios de entidades.
4. `SaveChangesAsync` confirma la unidad de trabajo.
5. Las cadenas con secretos no pertenecen al repositorio.

-----

## Para profundizar
<details><summary>¿Repositorio sobre EF Core?</summary>`DbContext` ya implementa patrones similares a unidad de trabajo y repositorio. Una abstracción adicional es útil si expresa lenguaje o límites del dominio; envolver cada método de EF sin aportar semántica solo agrega ceremonia.</details>

-----

## En entrevista
### Respuesta corta (junior)
`DbContext` conecta el modelo con la base, ejecuta consultas, rastrea cambios y guarda una unidad de trabajo.

### Respuesta ampliada (semi-senior)
Debe tener vida corta, normalmente scoped por request, y nunca compartirse entre hilos. El proveedor traduce LINQ y cambios al motor concreto; configuración y secretos se mantienen fuera del código.

### Preguntas frecuentes de seguimiento
**1. ¿`DbSet` es una tabla?** Es una abstracción de consulta y cambios; el mapeo determina su relación con el esquema.

**2. ¿`Add` ejecuta un INSERT?** No; normalmente ocurre al llamar `SaveChanges`.

-----

## Práctica
**Ejercicio 1.** Cambia el proveedor a SQL Server sin tocar la entidad.
<details><summary>Solución</summary>Agrega `Microsoft.EntityFrameworkCore.SqlServer`, reemplaza `UseSqlite` por `UseSqlServer` y proporciona una cadena válida desde configuración segura. Las migraciones deben revisarse para ese proveedor.</details>

-----

## Siguiente lección
[Migraciones y consultas](02-Migraciones%20y%20consultas.md)
