# CRUD seguro con MVC y EF Core

## En una frase

Un CRUD correcto recibe modelos de formulario (no entidades), valida en el servidor, carga la entidad y cambia solo los campos permitidos, protege cada POST con antiforgery, redirige después de guardar y trata como errores esperables los duplicados y los cambios simultáneos.

-----

## Antes de empezar

Conviene que ya sepas:

* Controladores, vistas, Tag Helpers y `ModelState`: [MVC, routing y vistas Razor](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/03-MVC%20routing%20y%20vistas%20Razor.md).
* Change tracker y consultas: [DbContext, entidades y configuración](01-DbContext%20entidades%20y%20configuracion.md) y [Migraciones y consultas](02-Migraciones%20y%20consultas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Overposting:** el cliente envía campos que el formulario no muestra y el servidor los guarda.
* **Antiforgery (CSRF):** protección que impide que otro sitio envíe formularios en nombre del usuario.
* **PRG (Post/Redirect/Get):** después de un POST exitoso, redirigir a un GET.
* **Concurrencia optimista:** detectar al guardar si otro usuario cambió la misma fila.

-----

## El problema

Un controlador "generado rápido":

```csharp
public sealed class ContactsController(AppDbContext db) : Controller
{
    [HttpPost]
    public async Task<IActionResult> Edit(Contact contact)
    {
        db.Update(contact);
        await db.SaveChangesAsync();
        return View(contact);
    }
}
```

La entidad tiene un campo que el formulario no muestra:

```csharp
public sealed class Contact
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public bool IsVip { get; set; }           // solo lo cambia un administrador
    public DateTime CreatedAt { get; set; }
}
```

Lo que pasa en producción:

* Un usuario agrega `IsVip=true` al formulario con las herramientas del navegador. El model binding lo enlaza y `Update` lo guarda: **overposting**.
* El formulario no envía `CreatedAt`. `Update` marca **todas** las columnas como modificadas y guarda `0001-01-01`.
* No hay `[ValidateAntiForgeryToken]`: una página de otro sitio puede enviar este formulario con la cookie de sesión de la víctima.
* Devuelve `View` después de guardar. El usuario presiona F5, el navegador pregunta "¿reenviar formulario?" y el POST se repite.
* Nadie revisa `ModelState`: un email inválido se guarda igual.

-----

## Cómo funciona

### 1. Modelo de formulario: solo los campos permitidos

```csharp
public sealed record ContactForm(
    [Required, StringLength(120)] string Name,
    [Required, EmailAddress, StringLength(254)] string Email);
```

El formulario no puede enviar `IsVip` ni `CreatedAt` porque **no existen en el tipo que se enlaza**. Es la defensa más simple contra el overposting.

En un record posicional, los atributos van **en el parámetro**, sin `property:`. MVC enlaza el record por su constructor y, si encuentra metadatos de validación en la propiedad, lanza `InvalidOperationException` ("metadata must be associated with the constructor parameter").

### 2. Validar antes de tocar la base

```csharp
if (!ModelState.IsValid)
{
    return View(form);
}
```

En MVC con vistas, la acción se ejecuta aunque la validación falle. Devolver la vista con el mismo modelo muestra los errores junto a cada campo (`asp-validation-for`). Los atributos validan la **forma** de la entrada; reglas como "el email no se repite" necesitan consultar la base.

### 3. Crear: construir la entidad explícitamente

```csharp
db.Contacts.Add(new Contact { Name = form.Name.Trim(), Email = email });
await db.SaveChangesAsync(cancellationToken);
```

Tú decides qué campos vienen del formulario y cuáles fija el sistema (`CreatedAt`, `IsVip = false`).

### 4. Editar: cargar, asignar, guardar

```text
❌ db.Update(entidadRecibida)        marca TODAS las columnas como modificadas
✔ cargar → asignar campos permitidos → SaveChanges   el change tracker solo actualiza lo que cambió
```

```csharp
var contact = await db.Contacts.FindAsync([id], cancellationToken);
if (contact is null) return NotFound();

contact.Name = form.Name.Trim();
contact.Email = form.Email.Trim();
await db.SaveChangesAsync(cancellationToken);
```

El `id` viene de la ruta, no del formulario. Si además el recurso tiene dueño, esta es la línea donde se comprueba que el usuario puede editarlo (autorización basada en recurso).

### 5. Antiforgery en cada POST

```csharp
[HttpPost, ValidateAntiForgeryToken]
```

`<form method="post">` con Tag Helpers agrega un campo oculto con el token. Un sitio externo no puede leerlo, así que su formulario falsificado llega sin token y recibe `400`. Puedes aplicarlo globalmente con `options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute())` en `AddControllersWithViews`.

### 6. Post/Redirect/Get

```text
POST /Contacts/Create ──► guarda ──► 302 Location: /
                                        │
navegador ──► GET / ◄───────────────────┘   F5 repite el GET, no el POST
```

Para mostrar "Contacto creado" después de redirigir, usa `TempData`: sobrevive exactamente a una redirección.

### 7. Errores esperables: duplicados y concurrencia

**Duplicados.** El índice único de `Email` (lección 1) protege la regla en la base. Comprueba antes con `AnyAsync` para dar un mensaje claro; atrapa además `DbUpdateException`, porque entre la comprobación y el `INSERT` otro usuario pudo guardar el mismo email.

**Concurrencia.** Dos personas abren el mismo contacto y guardan. Sin control, gana el último y el primero pierde su cambio sin enterarse. Un *token de concurrencia* lo detecta:

```csharp
public Guid Version { get; set; }                                   // en la entidad
contact.Property(c => c.Version).IsConcurrencyToken();              // en OnModelCreating
```

EF Core agrega `WHERE Version = @original` al `UPDATE`. Si otra persona cambió la fila, se actualizan 0 filas y EF Core lanza `DbUpdateConcurrencyException`. En SQL Server se usa una columna `rowversion` con `[Timestamp]`, que la base actualiza sola; en SQLite asignas un `Guid.NewGuid()` nuevo en cada guardado.

### 8. Borrar: POST y una sola sentencia

```csharp
await db.Contacts.Where(c => c.Id == id).ExecuteDeleteAsync(cancellationToken);
```

`ExecuteDeleteAsync` ejecuta un `DELETE` directo, sin cargar la entidad. Va en un POST con antiforgery, nunca en un enlace GET.

-----

## Ejemplo completo

Proyecto `dotnet new web` con las clases `Contact` y `AppDbContext` de la [lección 1](01-DbContext%20entidades%20y%20configuracion.md) (con el índice único en `Email`), la cadena de conexión y la migración de la [lección 2](02-Migraciones%20y%20consultas.md).

`Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
var connection = builder.Configuration.GetConnectionString("Database")
    ?? throw new InvalidOperationException("Falta ConnectionStrings:Database.");

builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlite(connection));
builder.Services.AddControllersWithViews();

var app = builder.Build();
app.MapControllerRoute("default", "{controller=Contacts}/{action=Index}/{id?}");
app.Run();

// Contact y AppDbContext: de la lección 1.
```

`Controllers/ContactsController.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

public sealed class ContactsController(AppDbContext db) : Controller
{
    public async Task<IActionResult> Index(CancellationToken cancellationToken) =>
        View(await db.Contacts
            .AsNoTracking()
            .OrderBy(c => c.Id)
            .Select(c => new ContactRow(c.Id, c.Name, c.Email))
            .ToListAsync(cancellationToken));

    [HttpGet]
    public IActionResult Create() => View(new ContactForm("", ""));

    [HttpPost, ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(ContactForm form, CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
        {
            return View(form);
        }

        var email = form.Email.Trim();
        if (await db.Contacts.AnyAsync(c => c.Email == email, cancellationToken))
        {
            ModelState.AddModelError(nameof(ContactForm.Email), "Ya existe un contacto con ese email.");
            return View(form);
        }

        db.Contacts.Add(new Contact { Name = form.Name.Trim(), Email = email });
        await db.SaveChangesAsync(cancellationToken);

        TempData["Mensaje"] = "Contacto creado.";
        return RedirectToAction(nameof(Index));
    }

    [HttpGet]
    public async Task<IActionResult> Edit(int id, CancellationToken cancellationToken)
    {
        var form = await db.Contacts
            .AsNoTracking()
            .Where(c => c.Id == id)
            .Select(c => new ContactForm(c.Name, c.Email))
            .SingleOrDefaultAsync(cancellationToken);

        return form is null ? NotFound() : View(form);
    }

    [HttpPost, ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(int id, ContactForm form, CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
        {
            return View(form);
        }

        var contact = await db.Contacts.FindAsync([id], cancellationToken);
        if (contact is null)
        {
            return NotFound();
        }

        contact.Name = form.Name.Trim();
        contact.Email = form.Email.Trim();

        try
        {
            await db.SaveChangesAsync(cancellationToken);
        }
        catch (DbUpdateException)
        {
            // Simplificado: en producción, distingue el código de error de unicidad del proveedor.
            ModelState.AddModelError(nameof(ContactForm.Email), "Ya existe un contacto con ese email.");
            return View(form);
        }

        TempData["Mensaje"] = "Contacto actualizado.";
        return RedirectToAction(nameof(Index));
    }

    [HttpPost, ValidateAntiForgeryToken]
    public async Task<IActionResult> Delete(int id, CancellationToken cancellationToken)
    {
        await db.Contacts.Where(c => c.Id == id).ExecuteDeleteAsync(cancellationToken);
        TempData["Mensaje"] = "Contacto eliminado.";
        return RedirectToAction(nameof(Index));
    }
}

public sealed record ContactForm(
    [Required, StringLength(120)] string Name,
    [Required, EmailAddress, StringLength(254)] string Email);

public sealed record ContactRow(int Id, string Name, string Email);
```

`Views/_ViewImports.cshtml`:

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

`Views/Contacts/Index.cshtml`:

```cshtml
@model IReadOnlyList<ContactRow>
<h1>Contactos</h1>
<p>@TempData["Mensaje"]</p>
<a asp-action="Create">Nuevo contacto</a>
<ul>
@foreach (var contact in Model)
{
    <li>
        @contact.Name (@contact.Email)
        <a asp-action="Edit" asp-route-id="@contact.Id">Editar</a>
        <form asp-action="Delete" asp-route-id="@contact.Id" method="post">
            <button type="submit">Eliminar</button>
        </form>
    </li>
}
</ul>
```

`Views/Contacts/Create.cshtml` (y `Edit.cshtml`, igual pero con `asp-action="Edit"`):

```cshtml
@model ContactForm
<h1>Nuevo contacto</h1>
<form asp-action="Create" method="post">
    <label asp-for="Name"></label>
    <input asp-for="Name" />
    <span asp-validation-for="Name"></span>

    <label asp-for="Email"></label>
    <input asp-for="Email" />
    <span asp-validation-for="Email"></span>

    <button type="submit">Guardar</button>
</form>
```

Comportamiento:

```text
POST /Contacts/Create  Name=Ana, Email=ana@example.com, token    → 302 Location: /
GET  /                                                            → 200 "Contacto creado." + Ana (ana@example.com)
POST /Contacts/Create  Name=Luis, Email=no-es-email, token        → 200 formulario con
                                                                    "The Email field is not a valid e-mail address."
POST /Contacts/Create  Name=Otra, Email=ana@example.com, token    → 200 formulario con
                                                                    "Ya existe un contacto con ese email."
POST /Contacts/Create  sin token antiforgery                      → 400
POST /Contacts/Edit/1  Name=Ana María, Email=..., IsVip=true, token → 302; IsVip se ignora
POST /Contacts/Delete/1  token                                    → 302 Location: /
```

Qué observar:

* `IsVip=true` no tiene dónde enlazarse: `ContactForm` no lo declara.
* El `Edit` POST cambia dos propiedades de una entidad cargada; el `UPDATE` incluye solo las columnas que cambiaron.
* `RedirectToAction(nameof(Index))` genera `Location: /` porque `Contacts/Index` son los valores por defecto de la ruta.
* `<form asp-action="Edit" method="post">` publica a `/Contacts/Edit/1`: el Tag Helper reutiliza el `id` de la ruta actual.

-----

## Errores comunes

**1. Recibir la entidad en la acción.**
Qué pasa: overposting de campos como `IsVip`, roles o saldos.
Por qué: el model binding llena todas las propiedades públicas que encuentre en la petición.
Arreglo: un modelo de formulario con solo los campos editables.

**2. Usar `[property: Required]` en un record posicional enlazado por MVC.**
Qué pasa: `InvalidOperationException` al enlazar: la metadata debe estar en el parámetro del constructor.
Por qué: MVC crea el record con su constructor y valida los parámetros.
Arreglo: `[Required]` directamente en el parámetro.

**3. Llamar a `Update` con un objeto desconectado.**
Qué pasa: se sobrescriben con valores por defecto las columnas que el formulario no envió.
Por qué: `Update` marca todas las propiedades como modificadas.
Arreglo: carga la entidad, asigna los campos permitidos y guarda.

**4. Omitir antiforgery.**
Qué pasa: otro sitio provoca acciones con la sesión del usuario (CSRF).
Por qué: el navegador envía la cookie de sesión automáticamente.
Arreglo: `[ValidateAntiForgeryToken]` en cada POST, o `AutoValidateAntiforgeryTokenAttribute` global.

**5. Devolver la vista después de guardar.**
Qué pasa: F5 repite el POST y crea duplicados.
Por qué: el navegador reenvía la última petición.
Arreglo: Post/Redirect/Get con `RedirectToAction` y `TempData` para el mensaje.

**6. Tratar un duplicado como un error 500.**
Qué pasa: el usuario ve una página de error genérica por un email repetido.
Por qué: la `DbUpdateException` del índice único no se maneja.
Arreglo: valida antes con `AnyAsync` y atrapa `DbUpdateException` para devolver un error en `ModelState`.

-----

## Según la versión de .NET

* **ASP.NET MVC 5 + EF6:** el mismo patrón sobre .NET Framework; el *scaffolding* generaba acciones con `[Bind]` para limitar campos.
* **ASP.NET Core 2.0:** los Tag Helpers de formulario agregan el token antiforgery automáticamente.
* **EF Core 7:** `ExecuteUpdateAsync` y `ExecuteDeleteAsync` para modificar sin cargar entidades.
* **.NET 8:** `UseAntiforgery()` para minimal APIs y Blazor; MVC sigue usando sus filtros.
* **.NET 10 / EF Core 10:** los principios no cambian: modelos de entrada explícitos, validación en el servidor, antiforgery y PRG.

-----

## Cuándo sí y cuándo no

**Usa MVC + EF Core directamente en el controlador cuando:** el flujo es un CRUD pequeño y las reglas caben en la validación del formulario.

**No dejes crecer el controlador cuando:** aparecen reglas de negocio, varias entidades por operación, integraciones o transacciones. Mueve esa lógica a casos de uso y al dominio; el controlador se queda traduciendo HTTP.

**Usa concurrencia optimista cuando:** dos personas pueden editar el mismo registro y perder un cambio importa (precios, stock, datos de clientes).

-----

## Resumen en 5 líneas

1. Los formularios enlazan modelos de entrada, no entidades: así no hay overposting.
2. `ModelState` se revisa antes de persistir; las reglas que dependen de la base se validan aparte.
3. Editar es cargar, asignar los campos permitidos y guardar; nunca `Update` con un objeto recibido.
4. Cada POST lleva antiforgery, y después de guardar se redirige (PRG).
5. Duplicados y cambios simultáneos son errores esperables: se muestran al usuario, no como 500.

-----

## Para profundizar

<details>
<summary>Validación frente a invariantes</summary>

`[EmailAddress]` comprueba la forma del texto. "El email no se repite" depende del estado de la base y de lo que hagan otros usuarios al mismo tiempo: por eso necesita el índice único además de la comprobación previa. Las reglas del dominio (un contacto VIP necesita un responsable asignado) no van en atributos: van en la entidad o en un caso de uso, para que se cumplan sin importar desde dónde se modifique el dato.

</details>

<details>
<summary>Responder a un conflicto de concurrencia</summary>

Al atrapar `DbUpdateConcurrencyException`, `ex.Entries` contiene las entidades en conflicto. Con `entry.GetDatabaseValuesAsync()` obtienes los valores actuales y puedes mostrarlos: "otro usuario cambió el email a X; ¿quieres sobrescribirlo?". Decidir automáticamente quién gana es una regla de negocio, no un detalle técnico.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso un modelo de formulario con solo los campos permitidos, reviso `ModelState`, guardo de forma asíncrona y redirijo después del POST. Cada formulario lleva token antiforgery.

### Respuesta ampliada (semi-senior)

Evito el overposting con modelos de entrada y edito cargando la entidad y asignando campos explícitos, para que el change tracker actualice solo lo que cambió. Protejo cada POST con antiforgery, aplico PRG con `TempData`, propago la cancelación y proyecto los listados con `AsNoTracking`. Trato los duplicados con un índice único más un manejo de `DbUpdateException`, y uso concurrencia optimista donde perder un cambio importa. Si las reglas crecen, el controlador delega en casos de uso.

### Preguntas frecuentes de seguimiento

**1. ¿Los atributos reemplazan las reglas del dominio?**
No. Validan la forma de la entrada; las invariantes viven en el dominio.

**2. ¿Por qué redirigir después de guardar?**
Para que refrescar repita un GET y no el POST.

**3. ¿Qué hace `Update` y por qué evitarlo con datos del formulario?**
Marca todas las propiedades como modificadas; sobrescribe las que el formulario no envió.

**4. ¿Cómo detectas que otro usuario cambió el registro?**
Con un token de concurrencia: EF Core lanza `DbUpdateConcurrencyException` si la fila cambió.

-----

## Práctica

**Ejercicio 1.** Agrega concurrencia optimista al `Edit` del ejemplo usando SQLite. Si hubo conflicto, muestra un error en el formulario.

<details>
<summary>Solución</summary>

En la entidad, `public Guid Version { get; set; }`. En `OnModelCreating`, `contact.Property(c => c.Version).IsConcurrencyToken();`, y una migración nueva. El formulario de edición debe llevar la versión que leyó:

```csharp
public sealed record EditContactForm(
    [Required, StringLength(120)] string Name,
    [Required, EmailAddress, StringLength(254)] string Email,
    Guid Version);
```

En el POST:

```csharp
var contact = await db.Contacts.FindAsync([id], cancellationToken);
if (contact is null) return NotFound();

db.Entry(contact).Property(c => c.Version).OriginalValue = form.Version;  // la que vio el usuario
contact.Name = form.Name.Trim();
contact.Email = form.Email.Trim();
contact.Version = Guid.NewGuid();

try
{
    await db.SaveChangesAsync(cancellationToken);
}
catch (DbUpdateConcurrencyException)
{
    ModelState.AddModelError("", "Otro usuario modificó este contacto. Recarga la página.");
    return View(form);
}
```

Asignar `OriginalValue` hace que el `UPDATE` incluya `WHERE Version = <la que vio el usuario>`. Si alguien guardó antes, no se actualiza ninguna fila y aparece el error. En la vista, agrega `<input asp-for="Version" type="hidden" />`.

</details>

**Ejercicio 2.** Un usuario dice: "Edité mi contacto y ahora la fecha de alta dice 01/01/0001". Mirando el controlador del problema, ¿qué línea lo causa y cómo la corriges?

<details>
<summary>Solución</summary>

`db.Update(contact)`. El objeto viene del formulario sin `CreatedAt`, así que vale `default(DateTime)`, y `Update` marca todas las columnas como modificadas. Corrección: cargar la entidad (`FindAsync`), asignar solo `Name` y `Email` desde un `ContactForm`, y guardar. `CreatedAt` nunca pasa por el formulario.

</details>

-----

## Siguiente lección

Terminaste el módulo. Vuelve al [índice](README.md). Los siguientes temas del bloque son relaciones, transacciones y concurrencia en profundidad.
