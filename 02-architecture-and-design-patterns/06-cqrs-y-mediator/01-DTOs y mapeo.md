# DTOs y mapeo

## En una frase

Un DTO (*Data Transfer Object*) es la forma pública de los datos que entran o salen de tu aplicación; existe para que el contrato de la API no sea una copia de tus tablas, y el mapeo entidad → DTO conviene hacerlo **dentro de la consulta** siempre que se pueda.

-----

## Antes de empezar

Conviene que ya sepas:

* Proyecciones y `AsNoTracking`: [Migraciones y consultas](../../04-data-access-and-transactions/01-entity-framework-core/04-Migraciones%20y%20consultas.md).
* Navegaciones y ciclos: [Relaciones y carga de datos](../../04-data-access-and-transactions/01-entity-framework-core/05-Relaciones%20y%20carga%20de%20datos.md).
* Records: [Structs y records](../../01-csharp-core-and-runtime/05-tipos-avanzados/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **DTO (*Data Transfer Object*):** objeto sin comportamiento que define la forma de los datos que cruzan un límite (HTTP, cola, otra capa).
* **Mapeo:** copiar datos de un tipo a otro, por ejemplo de `Usuario` a `UsuarioDto`.
* **Aplanar:** convertir un dato anidado en una propiedad simple (`Usuario.Rol.NombreRol` → `UsuarioDto.Rol`).
* **Contrato:** la forma de los datos que un cliente espera; cambiarla rompe a quien lo consume.
* **AutoMapper:** biblioteca que mapea por convención de nombres usando reflexión.

-----

## El problema

Un endpoint devuelve la entidad tal cual:

```csharp
app.MapGet("/usuarios", async (AppDbContext db) =>
    await db.Usuarios.Include(u => u.Rol).ToListAsync());
```

Lo que pasa:

1. **Fuga de datos.** `Usuario` tiene `PasswordHash`. El JSON lo incluye para cualquiera que llame al endpoint.
2. **Ciclos.** `Usuario.Rol` apunta al rol y `Rol.Usuarios` contiene al usuario. El serializador entra en un bucle y lanza `JsonException: A possible object cycle was detected`.
3. **Contrato atado al esquema.** Alguien renombra la columna `NombreRol` a `Nombre`. La app móvil, que leía `rol.nombreRol`, deja de mostrar el rol sin que nadie toque la API.

Otro equipo "lo arregla" con AutoMapper:

```csharp
var usuarios = await db.Usuarios.Include(u => u.Rol).ToListAsync();
return mapper.Map<List<UsuarioDto>>(usuarios);
```

El JSON ya no tiene el hash, pero la consulta sigue trayendo **todas** las columnas de las dos tablas, incluido el hash, las rastrea en el change tracker y recién después descarta lo que sobra.

Las consecuencias:

* datos sensibles expuestos;
* errores de serialización que aparecen al agregar una navegación;
* clientes que se rompen por cambios internos;
* consultas que mueven más datos de los que se devuelven.

-----

## Cómo funciona

### 1. Qué es (y qué no es) un DTO

```csharp
public sealed record UsuarioDto(int Id, string Email, string Rol);
```

* Solo datos: sin lógica, sin navegaciones, sin atributos de EF Core.
* Un `record` posicional es ideal: inmutable, con igualdad por valor y una línea.
* No es "la entidad sin algunos campos". Es el **contrato** de un caso de uso concreto: listar usuarios puede devolver `UsuarioDto` y el detalle, `UsuarioDetalleDto`.

### 2. DTOs de entrada y de salida

| | DTO de entrada | DTO de salida |
| --- | --- | --- |
| Ejemplo | `CrearUsuarioRequest(string Email, string Password)` | `UsuarioDto(int Id, string Email, string Rol)` |
| Protege contra | *Overposting*: el cliente no fija `Id`, `Rol` ni `PasswordHash` | Fugas de datos y ciclos |
| Validación | Sí | No hace falta |

Nunca reutilices el mismo tipo para ambos: el de entrada tiene `Password`, el de salida jamás.

### 3. Mapeo en la consulta (la opción por defecto)

```csharp
List<UsuarioDto> usuarios = await db.Usuarios
    .OrderBy(u => u.Email)
    .Select(u => new UsuarioDto(u.Id, u.Email, u.Rol!.NombreRol))
    .ToListAsync(ct);
```

```text
Map después de cargar                       Proyección en la consulta
SELECT u.*, r.*  (todas las columnas)       SELECT u.Id, u.Email, r.NombreRol
  → entidades rastreadas en memoria           → DTOs directamente
  → mapper copia 3 propiedades                (sin entidades, sin tracking)
```

EF Core traduce el `Select` a SQL: viajan tres columnas, no se crean entidades y no hay nada que rastrear. No necesitas `Include`; la navegación dentro del `Select` se convierte en un `JOIN`.

Para no repetir la proyección, guárdala como expresión:

```csharp
public static class UsuarioProyecciones
{
    public static readonly Expression<Func<Usuario, UsuarioDto>> ADto =
        u => new UsuarioDto(u.Id, u.Email, u.Rol!.NombreRol);
}

var usuarios = await db.Usuarios.Select(UsuarioProyecciones.ADto).ToListAsync(ct);
```

Tiene que ser `Expression<Func<...>>`, no `Func<...>`: EF Core necesita el árbol de expresión para traducirlo a SQL.

### 4. Mapeo en memoria: método de extensión

Cuando ya tienes la entidad (por ejemplo, después de crearla):

```csharp
public static class UsuarioMapeos
{
    public static UsuarioDto ADto(this Usuario usuario, string nombreRol) =>
        new(usuario.Id, usuario.Email, nombreRol);
}
```

Es explícito, el compilador te avisa si agregas un parámetro al DTO y **F12** te lleva al código del mapeo.

### 5. AutoMapper

```csharp
public sealed class UsuarioProfile : Profile
{
    public UsuarioProfile()
    {
        CreateMap<Usuario, UsuarioDto>()
            .ForMember(dest => dest.Rol, opt => opt.MapFrom(src => src.Rol!.NombreRol));
    }
}

builder.Services.AddAutoMapper(cfg => cfg.LicenseKey = builder.Configuration["AutoMapper:LicenseKey"], typeof(Program));
```

* `CreateMap` mapea solas las propiedades con el mismo nombre; `ForMember` resuelve las que no coinciden (aquí, aplanar `Rol.NombreRol`).
* Ese `UsuarioDto` debe ser una clase con propiedades `{ get; set; }`. Con un record posicional, AutoMapper usa el constructor y la configuración va con `ForCtorParam`, no con `ForMember`.
* Desde AutoMapper 13, el registro en DI viene en el paquete principal: el paquete `AutoMapper.Extensions.Microsoft.DependencyInjection` quedó obsoleto.
* Desde la versión 15 (2025), AutoMapper tiene **licencia comercial** para muchos usos: requiere una clave de licencia.
* Para consultas de EF Core usa `ProjectTo<UsuarioDto>(mapper.ConfigurationProvider)`, que genera el `Select`. `Map` después de `ToListAsync` repite el problema de traer todo.
* El `!` de `src.Rol!` solo calla al compilador. Si olvidaste cargar `Rol`, el `!` no lo carga.

### 6. Comparación honesta

| | Manual (`Select` / extensión) | AutoMapper | Generador de código (Mapperly) |
| --- | --- | --- | --- |
| Errores al renombrar | Error de compilación | En tiempo de ejecución, o propiedad en `null` silenciosa | Error o advertencia de compilación |
| Navegar al mapeo (F12) | Sí | No | Sí (código generado) |
| Proyección a SQL | Sí | Con `ProjectTo` | Sí, con proyecciones de `IQueryable` |
| Código que escribes | Más | Menos | Menos |
| Licencia | — | Comercial desde la v15 | MIT |

Si eliges AutoMapper, agrega una prueba que llame a `configuration.AssertConfigurationIsValid()`: detecta propiedades del destino que quedaron sin mapear.

### 7. ¿Dónde se mapea?

La regla que importa es una: **la entidad no cruza el límite de la API**. Dónde se escribe el mapeo depende de la arquitectura:

* API chica con Minimal APIs: en el endpoint, con un `Select`. Es perfectamente válido.
* Con capa de aplicación o handlers: en el handler de la consulta, que devuelve el DTO.

"Nunca en el controlador" es una preferencia de organización, no una ley. Lo que sí es un error es mapear después de traer entidades completas cuando podías proyectar.

-----

## Ejemplo completo

Consola con SQLite que compara serializar la entidad con serializar DTOs. `Dtos.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
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
using System.Linq.Expressions;
using System.Text.Json;
using Microsoft.EntityFrameworkCore;

var options = new DbContextOptionsBuilder<UsuariosContext>().UseSqlite("Data Source=dtos.db").Options;

await using (var db = new UsuariosContext(options))
{
    await db.Database.EnsureDeletedAsync();
    await db.Database.EnsureCreatedAsync();

    var admin = new Rol { NombreRol = "Admin" };
    var lector = new Rol { NombreRol = "Lector" };
    db.Usuarios.AddRange(
        new Usuario { Email = "ana@example.com", PasswordHash = "hash-de-ana", Rol = admin },
        new Usuario { Email = "luis@example.com", PasswordHash = "hash-de-luis", Rol = lector });
    await db.SaveChangesAsync();
}

var json = JsonSerializerOptions.Web;

// 1. Entidad sin Include: el hash viaja al cliente
await using (var db = new UsuariosContext(options))
{
    var ana = await db.Usuarios.SingleAsync(u => u.Email == "ana@example.com");
    Console.WriteLine($"Entidad:      {JsonSerializer.Serialize(ana, json)}");
}

// 2. Entidad con Include: Usuario -> Rol -> Usuarios -> Usuario...
await using (var db = new UsuariosContext(options))
{
    var ana = await db.Usuarios.Include(u => u.Rol).SingleAsync(u => u.Email == "ana@example.com");
    try
    {
        JsonSerializer.Serialize(ana, json);
    }
    catch (JsonException ex)
    {
        Console.WriteLine($"Con Include:  {ex.GetType().Name} (ciclo)");
    }
}

// 3. DTO proyectado en la consulta
await using (var db = new UsuariosContext(options))
{
    List<UsuarioDto> usuarios = await db.Usuarios
        .OrderBy(u => u.Email)
        .Select(UsuarioProyecciones.ADto)
        .ToListAsync();
    Console.WriteLine($"DTOs:         {JsonSerializer.Serialize(usuarios, json)}");
}

public sealed record UsuarioDto(int Id, string Email, string Rol);

public static class UsuarioProyecciones
{
    public static readonly Expression<Func<Usuario, UsuarioDto>> ADto =
        u => new UsuarioDto(u.Id, u.Email, u.Rol!.NombreRol);
}

public class Rol
{
    public int Id { get; set; }
    public required string NombreRol { get; set; }
    public List<Usuario> Usuarios { get; } = [];
}

public class Usuario
{
    public int Id { get; set; }
    public required string Email { get; set; }
    public required string PasswordHash { get; set; }
    public int RolId { get; set; }
    public Rol? Rol { get; set; }
}

public sealed class UsuariosContext(DbContextOptions<UsuariosContext> options) : DbContext(options)
{
    public DbSet<Usuario> Usuarios => Set<Usuario>();
    public DbSet<Rol> Roles => Set<Rol>();
}
```

Salida:

```text
Entidad:      {"id":1,"email":"ana@example.com","passwordHash":"hash-de-ana","rolId":1,"rol":null}
Con Include:  JsonException (ciclo)
DTOs:         [{"id":1,"email":"ana@example.com","rol":"Admin"},{"id":2,"email":"luis@example.com","rol":"Lector"}]
```

Qué observar:

* La entidad expone `passwordHash` y además `rolId` y `rol: null`, detalles internos que ningún cliente pidió.
* Con `Include`, el *fix-up* del change tracker llena `Rol.Usuarios` con el propio usuario: el ciclo aparece sin que escribas nada raro.
* El DTO trae exactamente lo que el cliente necesita, aplanado, y la consulta solo pidió esas columnas.
* `JsonSerializerOptions.Web` (.NET 9+) usa *camelCase*, igual que ASP.NET Core.

-----

## Errores comunes

**1. Devolver entidades desde la API.**
Qué pasa: fuga de campos sensibles, ciclos de serialización y contrato atado al esquema.
Por qué: el serializador recorre todas las propiedades públicas, incluidas las navegaciones.
Arreglo: un DTO por respuesta; la entidad no sale de la aplicación.

**2. Cargar entidades y mapear después.**
Qué pasa: la consulta trae todas las columnas y las rastrea, para devolver tres.
Por qué: `Map` (o un bucle con `new Dto`) trabaja en memoria, después del `SELECT`.
Arreglo: `Select` hacia el DTO, o `ProjectTo` si usas AutoMapper.

**3. Usar `Func<T, Dto>` en lugar de `Expression<Func<T, Dto>>` para reutilizar la proyección.**
Qué pasa: EF Core no puede traducirla y la consulta trae entidades completas (o lanza excepción, según dónde se use).
Por qué: un `Func` es código compilado; EF Core necesita un árbol de expresión que pueda leer.
Arreglo: `static readonly Expression<Func<Usuario, UsuarioDto>>`.

**4. Instalar `AutoMapper.Extensions.Microsoft.DependencyInjection` o `MediatR.Extensions.Microsoft.DependencyInjection`.**
Qué pasa: paquetes obsoletos que pueden chocar de versión con el paquete principal.
Por qué: AutoMapper 13 y MediatR 12 integraron el registro en DI.
Arreglo: instala solo `AutoMapper` (o `MediatR`) y revisa su licencia.

**5. Confiar en que AutoMapper "mapea todo".**
Qué pasa: tras renombrar `NombreRol`, la propiedad del DTO queda vacía sin error de compilación.
Por qué: el mapeo por convención se resuelve en tiempo de ejecución.
Arreglo: `AssertConfigurationIsValid()` en una prueba, o mapeo manual / generado en compilación.

-----

## Según la versión de .NET

* **C# 3:** tipos anónimos y árboles de expresión, la base de las proyecciones con `Select`.
* **C# 9:** `record` y `init`: DTOs inmutables en una línea.
* **C# 12:** constructores primarios en clases y expresiones de colección (`[]`).
* **.NET 9:** `JsonSerializerOptions.Web`.
* **AutoMapper 13 (2024):** DI integrada en el paquete principal. **AutoMapper 15 (2025):** licencia comercial.

-----

## Cuándo sí y cuándo no

**Usa DTOs cuando:** cruzas un límite: HTTP, mensajería, otra capa o un cliente que no controlas. Es decir, casi siempre en una API.

**Usa proyección manual cuando:** lees desde EF Core. Es la opción por defecto.

**Usa AutoMapper cuando:** tienes muchos mapeos triviales, el equipo ya lo conoce, aceptas su licencia y validas la configuración en pruebas.

**No crees DTOs cuando:** el tipo nunca sale del proceso; por ejemplo, un modelo interno entre dos métodos de la misma clase.

-----

## Resumen en 5 líneas

1. Un DTO es el contrato de un caso de uso, no la entidad sin algunos campos.
2. Separa DTOs de entrada (contra *overposting*) y de salida (contra fugas y ciclos).
3. Proyecta en la consulta con `Select`; reutiliza con `Expression<Func<...>>`.
4. AutoMapper ahorra código a cambio de errores en ejecución y licencia comercial desde la v15.
5. La entidad no cruza el límite de la API; dónde escribes el mapeo es secundario.

-----

## Para profundizar

<details>
<summary>DTOs y versionamiento de la API</summary>

Si el DTO es el contrato, un cambio incompatible (quitar o renombrar una propiedad) exige una nueva versión del DTO, no solo de la entidad. Agregar propiedades suele ser compatible: los clientes ignoran lo que no conocen. Por eso conviene que los DTOs de salida vivan junto a los endpoints y no en el proyecto del dominio. Ver [Versionamiento de APIs](../../03-aspnet-core-apis/01-diseno-de-apis-rest/03-Versionamiento%20de%20APIs.md).

</details>

<details>
<summary>¿Y <code>ReferenceHandler.IgnoreCycles</code>?</summary>

Hace que el serializador escriba `null` donde detecta un ciclo. Evita la excepción, pero sigues devolviendo la entidad con sus campos sensibles y un JSON cuya forma depende de qué navegaciones estaban cargadas. Es un parche del síntoma; el arreglo es el DTO.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un DTO es un objeto solo con datos que define qué entra y qué sale de la API. Lo uso para no exponer la entidad: evita filtrar campos como el hash de la contraseña y los ciclos al serializar. El mapeo lo hago con `Select` en la consulta.

### Respuesta ampliada (semi-senior)

Separo DTOs de entrada y de salida por caso de uso. Para lecturas proyecto en la consulta con `Select`, reutilizando expresiones cuando hace falta, así viajan solo las columnas necesarias y no hay tracking. Para mapeos en memoria uso métodos explícitos. Conozco AutoMapper: si se usa, `ProjectTo` para EF Core y `AssertConfigurationIsValid` en pruebas; tiene licencia comercial desde la v15, y alternativas como Mapperly detectan errores en compilación.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no devolver la entidad si tiene los mismos campos?**
Porque hoy coinciden; mañana la entidad tendrá un campo interno o una navegación, y el contrato cambiará sin que nadie lo decida.

**2. ¿Qué diferencia hay entre `Map` y `ProjectTo`?**
`Map` copia objetos ya cargados en memoria; `ProjectTo` genera el `Select` y la base devuelve solo las columnas del DTO.

**3. ¿El DTO debe tener validaciones?**
El de entrada sí (formato, obligatoriedad). El de salida no.

**4. ¿Dónde va el mapeo?**
Donde vive el caso de uso: el endpoint en una API simple, el handler si hay capa de aplicación.

-----

## Práctica

**Ejercicio 1.** Convierte este endpoint para que devuelva `TareaDto(Guid Id, string Titulo, string Categoria)` sin traer columnas de más.

```csharp
app.MapGet("/tareas", async (TareasContext db) =>
{
    var tareas = await db.Tareas.Include(t => t.Categoria).ToListAsync();
    return tareas.Select(t => new TareaDto(t.TareaId, t.Titulo, t.Categoria!.Nombre));
});
```

<details>
<summary>Solución</summary>

```csharp
app.MapGet("/tareas", async (TareasContext db, CancellationToken ct) =>
    TypedResults.Ok(await db.Tareas
        .OrderBy(t => t.Titulo)
        .Select(t => new TareaDto(t.TareaId, t.Titulo, t.Categoria!.Nombre))
        .ToListAsync(ct)));
```

El `Select` pasa **antes** de `ToListAsync`, así que se traduce a SQL. Ya no hace falta `Include` y no hay entidades rastreadas.

</details>

**Ejercicio 2.** Un compañero propone usar `Usuario` como modelo de entrada del registro "porque tiene todos los campos". Da tres razones concretas para no hacerlo y escribe el DTO correcto.

<details>
<summary>Solución</summary>

1. El cliente podría enviar `"rolId": 1` y registrarse como administrador (*overposting*).
2. `Usuario` tiene `PasswordHash`, no `Password`: el modelo no representa lo que el cliente manda.
3. Agregar una propiedad interna a la entidad la convierte automáticamente en un campo aceptado por la API.

```csharp
public sealed record RegistrarUsuarioRequest(string Email, string Password);
```

El servidor calcula el hash y asigna el rol por defecto.

</details>

-----

## Siguiente lección

[CQRS y mediator con MediatR](02-CQRS%20y%20mediator%20con%20MediatR.md)
