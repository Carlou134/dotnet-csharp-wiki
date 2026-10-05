# CRUD seguro con MVC y EF Core

## En una frase
Un CRUD correcto separa modelos HTTP de entidades, valida en el servidor, limita los campos modificables y mantiene explícitos los cambios y fallos de concurrencia.

-----

## Antes de empezar
Conviene que ya sepas: [MVC, routing y vistas Razor](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/03-MVC%20routing%20y%20vistas%20Razor.md) y [Migraciones y consultas](02-Migraciones%20y%20consultas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)): **overposting:** modificación de campos no autorizados; **PRG:** redirección después de un POST; **concurrencia optimista:** detección de cambios simultáneos al guardar.

-----

## El problema
Recibir una entidad completa desde un formulario y llamar `Update` permite que el cliente envíe propiedades que la pantalla no muestra. Además, devolver la misma respuesta después de guardar hace que actualizar el navegador pueda repetir el POST.

-----

## Cómo funciona
### 1. Contrato de entrada
```csharp
public sealed record CreateContactRequest(
    [Required, StringLength(120)] string Name,
    [Required, EmailAddress] string Email);
```
El contrato contiene solo campos permitidos. Los atributos ayudan al model binding, pero el dominio debe conservar invariantes propias.

En un record posicional, los atributos van **en el parámetro**, sin `property:`. MVC enlaza el record por su constructor y, si encuentra metadatos de validación en la propiedad, lanza `InvalidOperationException` ("metadata must be associated with the constructor parameter").

### 2. GET para leer, POST para cambiar
GET muestra el formulario. POST valida y persiste. Las eliminaciones usan POST o un verbo de API apropiado; nunca GET.

### 3. ModelState
En MVC con vistas, si `ModelState.IsValid` es falso, vuelve a la vista con el modelo para mostrar errores. No intentes persistir primero.

### 4. Crear explícitamente
Construye una entidad desde los campos aceptados. No vincules propiedades internas como `Id`, rol, saldo o fecha de auditoría.

### 5. Editar explícitamente
Carga la entidad por ID, comprueba que existe y asigna únicamente propiedades editables. `db.Update(request)` marca más estado del necesario y facilita overposting.

### 6. Post/Redirect/Get
Después de guardar, redirige a una acción GET. Así la URL representa el recurso y refrescar no reenvía el formulario.

### 7. Concurrencia
En datos sensibles, agrega un token de concurrencia y maneja `DbUpdateConcurrencyException`. Sin ese control, la última escritura puede ocultar la anterior.

-----

## Ejemplo completo
```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

public sealed class ContactsController(AppDbContext db) : Controller
{
    [HttpGet]
    public IActionResult Create() => View(new CreateContactRequest("", ""));

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(
        CreateContactRequest request,
        CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
        {
            return View(request);
        }

        var contact = new Contact
        {
            Name = request.Name.Trim(),
            Email = request.Email.Trim()
        };

        db.Contacts.Add(contact);
        await db.SaveChangesAsync(cancellationToken);

        return RedirectToAction(nameof(Details), new { id = contact.Id });
    }

    [HttpGet]
    public async Task<IActionResult> Details(int id, CancellationToken cancellationToken)
    {
        ContactRow? contact = await db.Contacts
            .AsNoTracking()
            .Where(x => x.Id == id)
            .Select(x => new ContactRow(x.Id, x.Name, x.Email))
            .SingleOrDefaultAsync(cancellationToken);

        return contact is null ? NotFound() : View(contact);
    }
}

public sealed record CreateContactRequest(
    [Required, StringLength(120)] string Name,
    [Required, EmailAddress] string Email);

public sealed record ContactRow(int Id, string Name, string Email);
```

Tras crear correctamente, la respuesta es una redirección HTTP hacia `Details`; el ID depende de la base y no se inventa una salida fija.

-----

## Errores comunes
**1. Recibir la entidad persistente.** Qué pasa: el cliente puede enviar campos adicionales. Por qué: model binding completa propiedades públicas. Arreglo: usa un request/ViewModel limitado.

**2. Omitir antiforgery en formularios.** Qué pasa: otro sitio puede provocar una acción con la sesión del usuario. Por qué: la cookie viaja con la solicitud. Arreglo: usa el token y `[ValidateAntiForgeryToken]`.

**3. Llamar `Update` con un objeto desconectado.** Qué pasa: se marcan muchas propiedades como modificadas. Por qué: el contexto no conoce su estado original. Arreglo: carga y cambia campos explícitos.

**4. Devolver la vista después de guardar.** Qué pasa: refrescar puede repetir el POST. Por qué: el navegador conserva la última solicitud. Arreglo: aplica Post/Redirect/Get.

-----

## Según la versión de .NET
- **ASP.NET MVC 5 + EF6:** usaba el stack clásico de .NET Framework.
- **ASP.NET Core + EF Core:** integra DI, endpoint routing y middleware del stack moderno.
- **.NET 10 / EF Core 10:** conserva los principios de binding explícito, async, antiforgery y concurrencia; usa paquetes de la misma generación.

-----

## Cuándo sí y cuándo no
**Usa MVC + EF Core directamente cuando:** el flujo es pequeño y la lógica de negocio es simple. **No dejes crecer el controlador cuando:** aparecen reglas, integraciones o transacciones complejas; delega en casos de uso y dominio.

-----

## Resumen en 5 líneas
1. Los formularios reciben modelos de entrada, no entidades completas.
2. Toda validación relevante se repite en el servidor.
3. Los cambios cargan la entidad y asignan campos permitidos.
4. Formularios mutables necesitan antiforgery y verbos correctos.
5. Post/Redirect/Get evita repetir envíos al refrescar.

-----

## Para profundizar
<details><summary>Validación frente a invariantes</summary>Los atributos validan forma de entrada. Una regla como «un email debe ser único» requiere consultar estado y manejar carreras; no se resuelve solo con `[EmailAddress]`.</details>

-----

## En entrevista
### Respuesta corta (junior)
Uso ViewModels para aceptar solo campos permitidos, valido `ModelState`, guardo de forma asíncrona y redirijo después del POST.

### Respuesta ampliada (semi-senior)
Evito overposting cargando la entidad y aplicando cambios explícitos. Protejo formularios con antiforgery, propago cancelación, proyecto modelos de lectura y manejo concurrencia según el riesgo del negocio.

### Preguntas frecuentes de seguimiento
**1. ¿Los atributos reemplazan reglas del dominio?** No; validan principalmente contratos de entrada.

**2. ¿Por qué redirigir después de guardar?** Para aplicar PRG y evitar reenvíos accidentales.

-----

## Práctica
**Ejercicio 1.** Diseña un `EditContactRequest` que no permita cambiar el ID.
<details><summary>Solución</summary>

```csharp
public sealed record EditContactRequest(
    [Required, StringLength(120)] string Name,
    [Required, EmailAddress] string Email);
```

Recibe el ID desde la ruta, carga la entidad y modifica solo `Name` y `Email`.
</details>

-----

## Siguiente lección
Terminaste el módulo. Vuelve al [índice](README.md).
