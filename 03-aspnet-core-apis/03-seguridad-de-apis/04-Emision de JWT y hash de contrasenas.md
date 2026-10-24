# Emisión de JWT y hash de contraseñas

## En una frase

Cuando tu propia API hace el login, guarda las contraseñas como **hash lento con sal** (BCrypt o PBKDF2), las verifica sin revelar qué usuarios existen y emite un JWT corto, firmado con una clave secreta que nunca entra al repositorio, con claims que la misma API sabe leer.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un access token y por qué un JWT no es secreto: [OAuth 2.0, OpenID Connect y tokens](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md).
* Cómo valida la API un token con `AddJwtBearer`: [Autenticación JWT bearer en ASP.NET Core](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md).
* Roles, claims y `401` frente a `403`: [Autorización con claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Hash de contraseña:** resultado de aplicar una función de un solo sentido a la contraseña; no se puede revertir.
* **Sal (*salt*):** valor aleatorio que se mezcla con cada contraseña antes del hash, para que dos contraseñas iguales den hashes distintos.
* **Factor de costo:** parámetro que hace el hash deliberadamente lento (BCrypt) para encarecer los ataques de fuerza bruta.
* **Clave simétrica:** el mismo secreto firma y verifica el token (HMAC, `HS256`).
* **Enumeración de usuarios:** descubrir qué emails están registrados por diferencias en las respuestas o en sus tiempos.

-----

## El problema

Una guía de clase arma el login así:

```csharp
// appsettings.json (en el repositorio)
"Jwt": { "Key": "TareasApi_JWT_Clave_Super_Segura_2025", "Issuer": "WebApi", "Audience": "WebApiUsers", "ExpireMinutes": 60 }

// Al generar el token
new Claim(ClaimTypes.NameIdentifier, usuario.NombreUsuario),

// En un BaseController
protected Guid GetCurrentUserId() => Guid.Parse(User.FindFirst(ClaimTypes.NameIdentifier)!.Value);
```

Lo que pasa:

1. **Cualquiera con acceso al repositorio puede fabricar tokens.** La clave está en `appsettings.json`: con ella se firma un JWT con `role = Admin` que la API acepta como legítimo. Borrarla del archivo no alcanza: queda en el historial de Git.
2. **`GetCurrentUserId` lanza `FormatException` en todas las peticiones.** El token guarda `NombreUsuario` ("carlos") en `NameIdentifier`, y el `BaseController` lo intenta convertir a `Guid`. La propia guía lista "FormatException: el ID no es Guid" como error común, sin ver que lo provoca su código.
3. **El nombre de usuario como identificador** cambia si el usuario lo edita, y entonces sus datos quedan asociados a un nombre viejo.
4. Si el login responde "usuario no existe" y "contraseña incorrecta" por separado, o tarda distinto en cada caso, un atacante puede averiguar qué emails están registrados.

Las consecuencias:

* tokens falsificables;
* un endpoint protegido que nunca funciona;
* información útil para un atacante en las respuestas del login.

-----

## Cómo funciona

### 1. Antes de emitir tokens propios

Las lecciones anteriores recomiendan un *authorization server* (Entra ID, Auth0, Keycloak, Duende). Emitir tus propios JWT es razonable cuando:

* es una API de primera parte, con sus propios usuarios y un solo cliente o pocos;
* no necesitas login federado, SSO ni delegación a terceros.

Si necesitas *refresh tokens*, revocación, MFA, recuperación de contraseña y bloqueo por intentos, estás reconstruyendo un proveedor de identidad: evalúa ASP.NET Core Identity o un proveedor antes de escribirlo a mano.

### 2. Guardar contraseñas: hash lento con sal

```csharp
string hash = BCrypt.Net.BCrypt.HashPassword("Ana-2026!");
// $2a$11$N9qo8uLOickgx2ZMRZoMye...   (algoritmo, costo 11, sal y hash en un solo string)

bool ok = BCrypt.Net.BCrypt.Verify("Ana-2026!", hash);   // true
```

| Función | ¿Para contraseñas? | Por qué |
| --- | --- | --- |
| Texto plano o cifrado reversible | ❌ | Quien obtenga la base (o la clave) tiene todas las contraseñas |
| SHA-256, MD5 | ❌ | Rapidísimas: una GPU prueba miles de millones por segundo |
| BCrypt (`BCrypt.Net-Next`) | ✔ | Lenta a propósito, con sal incluida y costo ajustable |
| PBKDF2 (`PasswordHasher<T>` de ASP.NET Core Identity) | ✔ | La que usa Identity; sin paquetes de terceros |
| Argon2id | ✔ | Recomendación actual de OWASP; requiere biblioteca externa |

* `Verify` extrae la sal y el costo del hash guardado y repite el cálculo. **Nunca** compares `HashPassword(intento) == hashGuardado`: cada llamada genera una sal nueva y el resultado nunca coincide.
* La propiedad del modelo se llama `PasswordHash`, no `Password`: el nombre recuerda qué hay guardado.
* BCrypt solo usa los primeros 72 bytes de la contraseña.

### 3. Verificar sin revelar qué usuarios existen

```csharp
Usuario? usuario = repositorio.BuscarPorEmail(request.Email);

// Verifica siempre contra un hash, exista o no el usuario: el tiempo de respuesta es parecido
bool valido = BCrypt.Net.BCrypt.Verify(request.Password, usuario?.PasswordHash ?? HashFicticio);

if (usuario is null || !valido)
{
    return TypedResults.Unauthorized();   // misma respuesta para ambos casos
}
```

Si el email no existe y respondes al instante, mientras que uno existente tarda los ~100 ms de BCrypt, la diferencia de tiempo delata qué emails están registrados.

### 4. La clave de firma

* **Fuera del repositorio:** *user secrets* en desarrollo (`dotnet user-secrets set "Jwt:Key" "..."`), variables de entorno o un gestor de secretos (Azure Key Vault, AWS Secrets Manager) en producción.
* **Larga y aleatoria:** para `HS256`, al menos 256 bits (32 bytes). Una frase legible como "TareasApi_JWT_Clave_Super_Segura_2025" tiene la longitud, pero poca aleatoriedad. Genérala, por ejemplo, con `openssl rand -base64 48`.
* **Valida al arrancar:** si falta o es corta, la aplicación no debe iniciar.

Con clave simétrica, quien puede verificar también puede firmar. Si otras APIs deben validar tus tokens, usa una clave asimétrica (RSA o ECDSA): firmas con la privada y publicas la pública.

### 5. Claims: identificadores estables y nombres cortos

```csharp
Claims = new Dictionary<string, object>
{
    ["sub"] = usuario.Id.ToString(),   // identificador estable: nunca cambia
    ["name"] = usuario.Nombre,         // para mostrar
    ["role"] = usuario.Rol             // para [Authorize(Roles = "Admin")] / RequireRole
}
```

```csharp
.AddJwtBearer(o =>
{
    o.MapInboundClaims = false;                 // conserva "sub", "name", "role" tal cual
    o.TokenValidationParameters = new TokenValidationParameters
    {
        ValidIssuer = opciones.Issuer,
        ValidAudience = opciones.Audience,
        IssuerSigningKey = clave,
        NameClaimType = "name",                 // User.Identity.Name
        RoleClaimType = "role"                  // IsInRole, RequireRole, [Authorize(Roles)]
    };
});
```

La confusión entre `ClaimTypes.Role` y `"role"` viene del **mapeo de claims**: por defecto, el handler traduce nombres cortos del JWT (`role`, `sub`) a URIs largas de .NET (`ClaimTypes.Role`, `ClaimTypes.NameIdentifier`). Con `MapInboundClaims = false` y `RoleClaimType` explícito, el nombre que emites es el que lees. Por defecto, la validación ya comprueba firma, emisor, audiencia y expiración.

### 6. Leer el usuario actual

```csharp
public static class ClaimsPrincipalExtensions
{
    public static Guid ObtenerUsuarioId(this ClaimsPrincipal usuario) =>
        Guid.TryParse(usuario.FindFirstValue("sub"), out Guid id)
            ? id
            : throw new InvalidOperationException("El token no tiene un 'sub' válido.");
}
```

Un método de extensión sobre `ClaimsPrincipal` sirve en controladores, Minimal APIs y servicios, sin obligar a heredar de un `BaseController`. Y falla con un mensaje claro en lugar de un `NullReferenceException` por el `!`.

Sobre `[Authorize]`: **no** es lo que llena `User`. El middleware de autenticación lo llena en cada petición si el token es válido. `[Authorize]` (o `RequireAuthorization`) es lo que **exige** que exista un usuario autenticado; sin él, la petición anónima también entra, con `User` vacío.

### 7. El token: corto y sin datos sensibles

* El *payload* se lee con cualquier decodificador base64: **no pongas** contraseñas, emails si no hacen falta, ni datos personales.
* Vida corta (15 a 60 minutos). Un JWT no se puede revocar antes de expirar sin infraestructura adicional.
* La respuesta del login es JSON: `{"accessToken": "...", "tokenType": "Bearer", "expiresIn": 3600}`.

-----

## Ejemplo completo

Proyecto `dotnet new web` con los paquetes `Microsoft.AspNetCore.Authentication.JwtBearer` 10.0.0 y `BCrypt.Net-Next` 4.0.3.

Configuración. En `appsettings.json` va lo que no es secreto:

```json
{
  "Jwt": {
    "Issuer": "https://tareas.example.com",
    "Audience": "tareas-api",
    "ExpireMinutes": 60
  }
}
```

La clave, con *user secrets* (en la carpeta del proyecto):

```text
dotnet user-secrets init
dotnet user-secrets set "Jwt:Key" "<al menos 32 caracteres aleatorios, por ejemplo de openssl rand -base64 48>"
```

`Program.cs`:

```csharp
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.IdentityModel.JsonWebTokens;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

var opciones = builder.Configuration.GetSection("Jwt").Get<JwtOptions>()
    ?? throw new InvalidOperationException("Falta la sección Jwt.");
var clave = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(opciones.Key));
if (clave.KeySize < 256)
{
    throw new InvalidOperationException("Jwt:Key debe tener al menos 32 bytes. Configúrala con user secrets.");
}

builder.Services.AddSingleton(new EmisorDeTokens(opciones, clave));
builder.Services.AddSingleton<RepositorioUsuarios>();

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.MapInboundClaims = false;
        o.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = opciones.Issuer,
            ValidAudience = opciones.Audience,
            IssuerSigningKey = clave,
            NameClaimType = "name",
            RoleClaimType = "role",
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
builder.Services.AddAuthorization();

var app = builder.Build();

app.MapPost("/auth/login", Results<Ok<LoginResponse>, UnauthorizedHttpResult> (
    LoginRequest request, RepositorioUsuarios usuarios, EmisorDeTokens emisor) =>
{
    Usuario? usuario = usuarios.BuscarPorEmail(request.Email);
    bool valido = BCrypt.Net.BCrypt.Verify(request.Password, usuario?.PasswordHash ?? RepositorioUsuarios.HashFicticio);

    if (usuario is null || !valido)
    {
        return TypedResults.Unauthorized();
    }
    return TypedResults.Ok(emisor.Emitir(usuario));
});

app.MapGet("/me", (ClaimsPrincipal user) => TypedResults.Ok(new
    {
        Id = user.ObtenerUsuarioId(),
        Nombre = user.Identity!.Name,
        Rol = user.FindFirstValue("role")
    }))
   .RequireAuthorization();

app.MapGet("/admin/reporte", () => TypedResults.Ok(new { Total = 2 }))
   .RequireAuthorization(p => p.RequireRole("Admin"));

app.Run();

public sealed class JwtOptions
{
    public string Key { get; set; } = "";
    public string Issuer { get; set; } = "";
    public string Audience { get; set; } = "";
    public int ExpireMinutes { get; set; } = 60;
}

public sealed record LoginRequest(string Email, string Password);
public sealed record LoginResponse(string AccessToken, string TokenType, int ExpiresIn);
public sealed record Usuario(Guid Id, string Nombre, string Email, string PasswordHash, string Rol);

public sealed class EmisorDeTokens(JwtOptions opciones, SymmetricSecurityKey clave)
{
    private readonly JsonWebTokenHandler _handler = new();

    public LoginResponse Emitir(Usuario usuario)
    {
        string token = _handler.CreateToken(new SecurityTokenDescriptor
        {
            Issuer = opciones.Issuer,
            Audience = opciones.Audience,
            Expires = DateTime.UtcNow.AddMinutes(opciones.ExpireMinutes),
            SigningCredentials = new SigningCredentials(clave, SecurityAlgorithms.HmacSha256),
            Claims = new Dictionary<string, object>
            {
                ["sub"] = usuario.Id.ToString(),
                ["name"] = usuario.Nombre,
                ["role"] = usuario.Rol
            }
        });
        return new LoginResponse(token, "Bearer", opciones.ExpireMinutes * 60);
    }
}

// Solo para el ejemplo: en una aplicación real los usuarios viven en la base de datos
public sealed class RepositorioUsuarios
{
    // Hash de una contraseña que nadie conoce: iguala el tiempo de respuesta cuando el email no existe
    public static readonly string HashFicticio = BCrypt.Net.BCrypt.HashPassword(Guid.NewGuid().ToString());

    private readonly List<Usuario> _usuarios =
    [
        new(Guid.Parse("6f9619ff-8b86-d011-b42d-00c04fc964ff"), "Ana", "ana@example.com", BCrypt.Net.BCrypt.HashPassword("Ana-2026!"), "Admin"),
        new(Guid.Parse("0f8fad5b-d9cb-469f-a165-70867728950e"), "Luis", "luis@example.com", BCrypt.Net.BCrypt.HashPassword("Luis-2026!"), "Lector")
    ];

    public Usuario? BuscarPorEmail(string email) =>
        _usuarios.FirstOrDefault(u => string.Equals(u.Email, email, StringComparison.OrdinalIgnoreCase));
}

public static class ClaimsPrincipalExtensions
{
    public static Guid ObtenerUsuarioId(this ClaimsPrincipal usuario) =>
        Guid.TryParse(usuario.FindFirstValue("sub"), out Guid id)
            ? id
            : throw new InvalidOperationException("El token no tiene un 'sub' válido.");
}
```

Peticiones (el token cambia en cada login; aquí se abrevia):

```text
POST /auth/login  {"email":"ana@example.com","password":"Ana-2026!"}
→ 200 {"accessToken":"eyJ...","tokenType":"Bearer","expiresIn":3600}

POST /auth/login  {"email":"ana@example.com","password":"otra"}       → 401
POST /auth/login  {"email":"nadie@example.com","password":"otra"}     → 401  (misma respuesta)

GET /me                                                               → 401
GET /me            Authorization: Bearer <token de Ana>
→ 200 {"id":"6f9619ff-8b86-d011-b42d-00c04fc964ff","nombre":"Ana","rol":"Admin"}

GET /admin/reporte Authorization: Bearer <token de Luis>             → 403
GET /admin/reporte Authorization: Bearer <token de Ana>              → 200 {"total":2}
```

Qué observar:

* La clave no aparece en ningún archivo del repositorio, y la aplicación se niega a arrancar si falta.
* Contraseña incorrecta y email inexistente responden igual, y los dos pasan por `BCrypt.Verify`.
* `sub` lleva el `Guid` del usuario, así que `ObtenerUsuarioId` funciona; `name` y `role` se leen con los mismos nombres con que se emitieron, gracias a `MapInboundClaims = false` y `NameClaimType`/`RoleClaimType`.
* Luis está autenticado pero no es Admin: `403`, no `401`.
* Si pegas el token en un decodificador de JWT, ves `sub`, `name`, `role`, `iss`, `aud` y `exp` en texto claro: por eso no lleva datos sensibles.

-----

## Errores comunes

**1. La clave de firma en `appsettings.json`.**
Qué pasa: cualquiera con acceso al repositorio (o a su historial) puede emitir tokens válidos con cualquier rol.
Por qué: la clave simétrica firma y verifica.
Arreglo: user secrets o variables de entorno; rota la clave si alguna vez se subió.

**2. Identificador inestable o de otro tipo en el claim.**
Qué pasa: `FormatException` al hacer `Guid.Parse` sobre un nombre de usuario, o datos asociados a un nombre que cambió.
Por qué: el claim de identidad lleva un dato que no es el identificador.
Arreglo: `sub` con el id inmutable del usuario; nombre en `name`.

**3. Comparar hashes o usar SHA-256 para contraseñas.**
Qué pasa: login que nunca valida (sal distinta en cada hash), o contraseñas que se crackean en minutos si se filtra la base.
Por qué: `HashPassword` usa una sal nueva cada vez; SHA-256 es demasiado rápida.
Arreglo: `BCrypt.Verify` (o `PasswordHasher<T>.VerifyHashedPassword`) sobre el hash guardado.

**4. Mensajes o tiempos distintos para "no existe" y "contraseña incorrecta".**
Qué pasa: enumeración de usuarios.
Por qué: la respuesta filtra información sobre la base.
Arreglo: misma respuesta y verificación contra un hash ficticio cuando el usuario no existe.

**5. Confusión entre `ClaimTypes.Role` y `"role"`.**
Qué pasa: `[Authorize(Roles = "Admin")]` responde `403` a un administrador.
Por qué: el mapeo de claims entrantes y el `RoleClaimType` configurado no coinciden con el nombre emitido.
Arreglo: `MapInboundClaims = false` y `RoleClaimType = "role"`, y emitir `"role"`.

**6. Creer que `[Authorize]` llena `User`.**
Qué pasa: se agregan `[Authorize]` "para que funcione `User`" donde la petición debería poder ser anónima, o se deja un endpoint sin protección creyendo que `User` vacío lo bloquea.
Por qué: la autenticación ocurre en el middleware; `[Authorize]` solo exige el resultado.
Arreglo: protege con `[Authorize]`/`RequireAuthorization` los endpoints que lo necesitan; en los opcionales, comprueba `User.Identity?.IsAuthenticated`.

-----

## Según la versión de .NET

* **ASP.NET Core 1.0:** `JwtBearer` con `JwtSecurityTokenHandler`.
* **ASP.NET Core 7:** `dotnet user-jwts` para tokens de desarrollo; `RequireAuthorization` con políticas en línea.
* **ASP.NET Core 8:** el handler JWT bearer usa `JsonWebTokenHandler` (más rápido) en lugar de `JwtSecurityTokenHandler`; `MapIdentityApi` agrega endpoints de login de ASP.NET Core Identity.
* **.NET 10:** `Microsoft.AspNetCore.Authentication.JwtBearer` sigue siendo el handler estándar; `JwtSecurityTokenHandler` se mantiene por compatibilidad.

-----

## Cuándo sí y cuándo no

**Emite tus propios JWT cuando:** la API tiene sus propios usuarios, un cliente de primera parte y no necesitas SSO, federación ni delegación.

**Usa ASP.NET Core Identity cuando:** necesitas gestión completa de usuarios (registro, confirmación de email, recuperación de contraseña, bloqueo, MFA) sin un proveedor externo.

**Usa un authorization server cuando:** hay varios clientes o APIs, login con proveedores externos, SSO o requisitos de cumplimiento. No reimplementes OAuth.

-----

## Resumen en 5 líneas

1. Contraseñas con hash lento y sal (BCrypt, PBKDF2, Argon2id), verificadas con `Verify`.
2. Mismo resultado y mismo trabajo para "no existe" y "contraseña incorrecta".
3. Clave de firma larga, aleatoria y fuera del repositorio; la app no arranca sin ella.
4. `sub` con el id estable; `MapInboundClaims = false` y `RoleClaimType` para leer lo que emitiste.
5. Tokens cortos, sin datos sensibles; `[Authorize]` exige, el middleware autentica.

-----

## Para profundizar

<details>
<summary>Refresh tokens</summary>

Con access tokens de 15 minutos, el cliente necesita renovarlos sin pedir la contraseña otra vez. Un *refresh token* es un valor aleatorio largo, guardado (hasheado) en la base con su expiración y su usuario, que se canjea por un nuevo access token. Debe rotar en cada uso (el viejo se invalida) y detectarse la reutilización de uno ya usado, que indica robo. Es exactamente el tipo de lógica que un proveedor de identidad ya resuelve: piénsalo antes de implementarlo.

</details>

<details>
<summary><code>PasswordHasher&lt;T&gt;</code> sin usar todo Identity</summary>

```csharp
var hasher = new PasswordHasher<Usuario>();
string hash = hasher.HashPassword(usuario, "Ana-2026!");
PasswordVerificationResult r = hasher.VerifyHashedPassword(usuario, hash, "Ana-2026!");
// Success, Failed o SuccessRehashNeeded (el hash usa parámetros viejos: conviene regenerarlo)
```

Está en `Microsoft.AspNetCore.Identity` (incluido en el framework compartido de ASP.NET Core) y usa PBKDF2 con iteraciones que aumentan entre versiones. `SuccessRehashNeeded` permite actualizar hashes viejos de forma transparente en el siguiente login.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Guardo las contraseñas con BCrypt y las verifico con `Verify`. En el login, si son correctas, genero un JWT firmado con una clave secreta, con el id del usuario y su rol como claims y una expiración corta. La API lo valida con `AddJwtBearer` y protejo endpoints con `[Authorize]`.

### Respuesta ampliada (semi-senior)

Uso hash lento con sal (BCrypt o el `PasswordHasher` de Identity), verifico siempre contra un hash aunque el usuario no exista y respondo igual en ambos casos para evitar enumeración. La clave de firma vive en un gestor de secretos, tiene al menos 256 bits y valido su presencia al arrancar. Emito `sub` con el id inmutable, `name` y `role`, desactivo el mapeo de claims entrantes y configuro `NameClaimType` y `RoleClaimType` para que lo emitido coincida con lo leído. Tokens de vida corta, sin datos sensibles. Si el sistema necesita refresh tokens, MFA o SSO, prefiero Identity o un authorization server antes que reimplementarlo.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no SHA-256 para contraseñas?**
Porque es rápida: permite probar miles de millones de contraseñas por segundo si se filtra la base.

**2. ¿Por qué `Verify` y no comparar hashes?**
Porque cada hash lleva una sal aleatoria; dos hashes de la misma contraseña son distintos.

**3. ¿Dónde guardas la clave del JWT?**
En user secrets en desarrollo y en variables de entorno o un gestor de secretos en producción, nunca en el repositorio.

**4. ¿`[Authorize]` es lo que llena `User`?**
No: lo llena el middleware de autenticación. `[Authorize]` exige que haya un usuario autenticado.

-----

## Práctica

**Ejercicio 1.** Agrega `POST /auth/registro` que reciba email, nombre y contraseña, rechace emails repetidos con `409` y guarde el usuario con rol `Lector`.

<details>
<summary>Solución</summary>

```csharp
public sealed record RegistroRequest(string Email, string Nombre, string Password);

app.MapPost("/auth/registro", Results<Created, Conflict, ValidationProblem> (
    RegistroRequest request, RepositorioUsuarios usuarios) =>
{
    if (request.Password.Length < 12)
    {
        return TypedResults.ValidationProblem(new Dictionary<string, string[]>
        {
            [nameof(request.Password)] = ["La contraseña debe tener al menos 12 caracteres."]
        });
    }
    if (usuarios.BuscarPorEmail(request.Email) is not null)
    {
        return TypedResults.Conflict();
    }

    var usuario = new Usuario(Guid.CreateVersion7(), request.Nombre.Trim(), request.Email.Trim(),
        BCrypt.Net.BCrypt.HashPassword(request.Password), "Lector");
    usuarios.Agregar(usuario);
    return TypedResults.Created($"/usuarios/{usuario.Id}");
});
```

(Requiere un método `Agregar` en el repositorio; con una base real, el índice único sobre `Email` es la garantía contra duplicados concurrentes.) El rol lo decide el servidor: el request no tiene un campo `Rol`. Responder `409` revela que el email existe; en sistemas sensibles, el registro responde siempre igual y avisa por email.

</details>

**Ejercicio 2.** Un compañero emite el token con `JsonWebTokenHandler` y `Subject = new ClaimsIdentity([new Claim(ClaimTypes.Role, "Admin")])`. La API valida con `MapInboundClaims = false` y `RoleClaimType = "role"`, y `RequireRole("Admin")` responde `403` a un administrador. ¿Qué está pasando y cómo lo diagnosticas?

<details>
<summary>Solución</summary>

`ClaimTypes.Role` no es `"role"`: es la constante `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`. `JsonWebTokenHandler` escribe el tipo del claim tal cual, así que el token lleva ese URI largo como nombre. La API, con el mapeo de entrada desactivado, busca un claim llamado `"role"`, no lo encuentra y el usuario no tiene rol: `403`.

Para diagnosticarlo, decodifica el token (cualquier decodificador de JWT) y mira el nombre real del claim. El arreglo es emitir `"role"` explícitamente (como en el ejemplo, con el diccionario `Claims`). Usar nombres cortos en ambos lados evita depender de mapeos implícitos, que además se comportan distinto entre `JwtSecurityTokenHandler` (que sí traduce los URIs largos a nombres cortos al escribir) y `JsonWebTokenHandler`.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Ejercicios de seguridad](Ejercicios.md) y vuelve al [índice del módulo](README.md).
