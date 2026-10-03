# Ejercicios: seguridad de APIs

Requisitos previos: [OAuth 2.0 y OpenID Connect](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md), [JWT bearer](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md) y [Claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md). Los ejercicios usan `dotnet new web` y `Microsoft.AspNetCore.Authentication.JwtBearer`.

-----

## Ejercicios guiados

### Ejercicio 1. Configurar autenticación JWT local

**Objetivo:** validar access tokens sin depender de un servidor demo externo.

**Contexto:** la guía usaba `https://demo.identityserver.io` con audience genérica. Un demo público puede cambiar o desaparecer y no representa una configuración controlada. Para desarrollo, .NET ofrece `dotnet user-jwts`.

**Instrucciones:** registra JWT bearer, genera un token local y comprueba `401` y `200`.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();
app.MapGet("/private", () => "authenticated")
    .RequireAuthorization();
app.Run();
```

```text
dotnet user-jwts create
GET /private                                      → 401
GET /private Authorization: Bearer <token válido> → 200 authenticated
```

</details>

**Qué observar:** `dotnet user-jwts` configura issuer, audience y firma para desarrollo. No se usa como emisor de producción.

### Ejercicio 2. Proteger endpoints con una policy

**Objetivo:** diferenciar autenticación de permiso para una acción.

**Contexto:** la guía aplicaba únicamente `RequireAuthorization()`. Eso permite cualquier identidad autenticada, aunque no tenga permiso para leer el recurso.

**Instrucciones:** exige `orders.read` y compara respuestas sin token, sin scope y con scope.

<details>
<summary>Solución</summary>

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("OrdersRead", policy => policy.RequireAssertion(context =>
        HasScope(context.User, "orders.read")));

var app = builder.Build();
app.MapGet("/orders", () => new[] { 42, 84 })
    .RequireAuthorization("OrdersRead");
app.Run();

static bool HasScope(ClaimsPrincipal user, string expected) =>
    user.FindAll("scope")
        .SelectMany(c => c.Value.Split(' ', StringSplitOptions.RemoveEmptyEntries))
        .Contains(expected, StringComparer.Ordinal);
```

```text
dotnet user-jwts create --scope orders.read
sin token                 → 401
token sin orders.read     → 403
token con orders.read     → 200 [42,84]
```

</details>

**Qué observar:** token válido y permiso suficiente son dos comprobaciones distintas.

### Ejercicio 3. Leer claims sin depender de Name

**Objetivo:** obtener un identificador estable del sujeto autenticado.

**Contexto:** la guía leía `user.Identity?.Name`, que puede ser null si el proveedor no emite o no mapea el claim configurado como nombre. El nombre visible tampoco es un identificador estable.

**Instrucciones:** devuelve `sub`, con compatibilidad para el mapeo de claims de .NET.

<details>
<summary>Solución</summary>

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();

app.MapGet("/profile", (ClaimsPrincipal user) =>
{
    var subject = user.FindFirst("sub")?.Value
        ?? user.FindFirst(ClaimTypes.NameIdentifier)?.Value
        ?? throw new InvalidOperationException("El token no contiene un sujeto.");

    return TypedResults.Ok(new { Subject = subject });
})
.RequireAuthorization();

app.Run();
```

Salida con un token válido:

```text
GET /profile → 200 {"subject":"<sujeto emitido>"}
```

El valor depende del token real y no se inventa.

</details>

**Qué observar:** `sub` identifica al sujeto dentro del issuer. Para una clave global debes considerar la pareja `(iss, sub)`.

### Ejercicio 4. Roles y políticas

**Objetivo:** aplicar un role sin convertirlo en la única regla de acceso.

**Contexto:** la guía usaba `AdminOnly`, pero no explicaba que el claim de roles depende del proveedor ni que los roles amplios suelen ser peores que policies por capacidad.

**Instrucciones:** crea una policy administrativa y prueba el contraste entre `401`, `403` y `200`.

<details>
<summary>Solución</summary>

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy => policy.RequireRole("admin"));

var app = builder.Build();
app.MapGet("/admin", () => "admin")
    .RequireAuthorization("AdminOnly");

app.Run();
```

```text
dotnet user-jwts create --role admin
sin token          → 401
token sin admin    → 403
token con admin    → 200 admin
```

</details>

**Qué observar:** en un proveedor real debes confirmar el nombre del claim y configurar `RoleClaimType`. Para acciones específicas, prefiere policies como `OrdersDelete`.

### Ejercicio 5. Integrar un proveedor externo

**Objetivo:** validar access tokens emitidos para una API registrada en un proveedor.

**Contexto:** la guía presentaba Auth0 como “autenticación externalizada y segura”. Faltaban configuración por entorno, permisos y la aclaración de que la API valida **access tokens**, no realiza el login OIDC.

**Instrucciones:** carga Authority y Audience desde configuración y exige un permiso.

<details>
<summary>Solución</summary>

`appsettings.json`:

```json
{
  "Authentication": {
    "Authority": "https://TU-DOMINIO.auth0.com/",
    "Audience": "https://orders-api.example"
  }
}
```

`Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);
var authority = builder.Configuration["Authentication:Authority"]
    ?? throw new InvalidOperationException("Falta Authentication:Authority.");
var audience = builder.Configuration["Authentication:Audience"]
    ?? throw new InvalidOperationException("Falta Authentication:Audience.");

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = authority;
        options.Audience = audience;
        options.MapInboundClaims = false;
    });

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("OrdersRead", policy =>
        policy.RequireClaim("permissions", "read:orders"));

var app = builder.Build();
app.MapGet("/orders", () => new[] { 42, 84 })
    .RequireAuthorization("OrdersRead");
app.Run();
```

Comportamiento después de registrar la API y obtener un access token real:

```text
token ausente o inválido               → 401
token válido sin read:orders           → 403
token para esta audience con permiso   → 200 [42,84]
```

</details>

**Qué observar:** Authority y Audience no son secretos, pero pertenecen a la configuración del entorno. El formato exacto del permission claim debe coincidir con el proveedor.

-----

## Retos

### Reto 1. API segura completa

**Misión:** crea un grupo `/api/orders` protegido por fallback o policy, deja `/health` anónimo, agrega `orders.read` y `orders.write`, y devuelve 401/403 correctamente.

**Pista:** protege por defecto y marca solo el endpoint público con `AllowAnonymous()`.

<details>
<summary>Solución</summary>

Registra JWT bearer y dos policies de scope. Configura una fallback policy con `RequireAuthenticatedUser()`. Mapea `/health` con `.AllowAnonymous()`, `GET /api/orders` con `OrdersRead` y `POST /api/orders` con `OrdersWrite`. Prueba al menos: sin token, token con un solo scope y token con ambos. No escribas lógica de permisos dentro de los handlers.

</details>

### Reto 2. Autorización basada en recurso

**Misión:** impide que un usuario con `orders.read` consulte pedidos de otro usuario, pero permite acceso con un permiso administrativo explícito.

**Pista:** compara el `(iss, sub)` autenticado con el propietario cargado desde almacenamiento; no aceptes OwnerId del cuerpo como prueba.

<details>
<summary>Solución</summary>

Crea un `AuthorizationHandler` que reciba el pedido como recurso. Autoriza cuando el sujeto coincide con el propietario o existe `orders.read.all`. En el endpoint carga primero el pedido, llama a `IAuthorizationService.AuthorizeAsync(user, order, policyName)` y devuelve `Forbid` cuando falle. Un token válido con `orders.read` pero dueño distinto debe producir 403.

</details>

### Reto 3. Evaluar un identity provider

**Misión:** compara Auth0, Duende IdentityServer y el proveedor corporativo disponible antes de elegir.

**Pista:** no compares solo features; incluye operación, residencia de datos, SLA, salida del proveedor, licencia, rotación y respuesta ante incidentes.

<details>
<summary>Solución</summary>

Documenta una matriz con: protocolos y flujos necesarios, gestión de usuarios/MFA, disponibilidad, claves y rotación, auditoría, cumplimiento, costo por usuarios/clientes, soporte, portabilidad y esfuerzo operativo. Auth0 reduce operación pero introduce dependencia SaaS; Duende da control y exige operación propia más licencia de producción salvo Community Edition calificada; un proveedor corporativo puede integrar el directorio existente pero acoplarte a su plataforma. La decisión debe responder al contexto, no a una preferencia genérica.

</details>

-----

## Checkpoint

1. ¿Por qué OAuth 2.0 no es por sí solo un protocolo de login?
2. ¿Qué token debe recibir una API: access token o ID token?
3. ¿Qué cuatro validaciones mínimas necesita un JWT bearer?
4. ¿Cuál es la diferencia operacional entre 401 y 403?
5. ¿Por qué `RequireAuthorization()` puede ser insuficiente?
6. ¿Qué responsabilidad de seguridad sigue perteneciendo a la API aunque uses Auth0 o Duende?
