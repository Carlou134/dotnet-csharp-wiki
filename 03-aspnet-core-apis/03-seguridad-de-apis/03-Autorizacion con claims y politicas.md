# Autorización con claims y políticas

## En una frase

Autenticar un token establece una identidad; autorizar significa comprobar, mediante políticas y el recurso concreto, si esa identidad puede realizar la acción solicitada.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo se valida un access token: [Autenticación JWT bearer](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md).
* Cómo agrupar rutas: [Endpoints y grupos](../02-minimal-apis/01-Endpoints%20y%20grupos%20de%20rutas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Claim:** afirmación emitida sobre el sujeto o el cliente.
* **Scope:** permiso delegado concedido para una API.
* **Role:** pertenencia a una categoría funcional.
* **Policy:** conjunto nombrado de requisitos de autorización.

-----

## El problema

Este endpoint está autenticado, pero no necesariamente protegido correctamente:

```csharp
app.MapDelete("/orders/{id:int}", DeleteOrder)
   .RequireAuthorization();
```

Cualquier token válido puede llegar al handler: un usuario de solo lectura, una aplicación de reportes o el dueño de otro pedido. `RequireAuthorization()` sin política responde únicamente “existe una identidad autenticada”.

La guía afirma que un token válido deja pasar “solo usuarios autenticados y autorizados”. Falta definir **autorizados para qué acción y sobre qué recurso**.

-----

## Cómo funciona

### 1. Claims son datos; policies expresan decisiones

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("OrdersRead", policy =>
        policy.RequireClaim("permission", "orders:read"))
    .AddPolicy("OrdersDelete", policy =>
        policy.RequireClaim("permission", "orders:delete"));
```

```csharp
app.MapGet("/orders", GetOrders).RequireAuthorization("OrdersRead");
app.MapDelete("/orders/{id:int}", DeleteOrder).RequireAuthorization("OrdersDelete");
```

La política nombra una capacidad de la aplicación. Evita repetir strings y reglas dispersas en cada handler.

### 2. Un scope puede venir como lista separada por espacios

En OAuth, `scope` suele contener `orders.read orders.write` en un solo claim. `RequireClaim("scope", "orders.read")` exige un valor exacto y puede no reconocer esa lista según cómo el proveedor materialice los claims.

```csharp
static bool HasScope(ClaimsPrincipal user, string expected) =>
    user.FindAll("scope")
        .SelectMany(claim => claim.Value.Split(
            ' ', StringSplitOptions.RemoveEmptyEntries))
        .Contains(expected, StringComparer.Ordinal);
```

Usa un requirement/handler reutilizable en aplicaciones reales. Una `RequireAssertion` pequeña sirve para mostrar la idea.

### 3. Roles y permisos no son lo mismo

Un role agrupa responsabilidades organizativas (`admin`, `support`). Un permission o scope expresa una acción (`orders.read`). Basar todo en `Admin` crea privilegios amplios y difíciles de auditar.

Además, el nombre del claim de roles depende del proveedor. Configura `RoleClaimType` o transforma claims conscientemente; no asumas que `RequireRole("Admin")` encontrará `roles`, `role` o una URI propietaria indistintamente.

### 4. 401 y 403 cuentan historias distintas

```text
sin token / token inválido        → 401 Unauthorized + challenge
token válido, política incumplida → 403 Forbidden
```

No devuelvas `404` o `200` para esconder sistemáticamente una configuración rota. Un recurso individual puede usar `404` para evitar enumeración, pero esa decisión debe formar parte del modelo de amenazas y ser consistente.

### 5. Algunas decisiones necesitan el recurso

El claim `orders.read` permite leer pedidos en general, pero no prueba que el pedido 42 pertenezca al usuario. Carga el recurso y evalúa una policy con `IAuthorizationService.AuthorizeAsync(user, order, policyName)`. Eso evita confiar en un `userId` enviado por el cliente.

### 6. Elegir proveedor es una decisión operativa

| Opción | Ventaja | Costo/riesgo |
| --- | --- | --- |
| Auth0 u otro SaaS | Menos infraestructura de identidad | Precio, dependencia y límites del proveedor |
| Duende IdentityServer | Control y despliegue propio | Operación crítica y licencia de producción |
| Proveedor cloud corporativo | Integración con directorio existente | Acoplamiento al ecosistema y configuración |

IdentityServer4 ya no es la opción mantenida. Su sucesor es **Duende IdentityServer**, que requiere licencia para producción salvo ediciones comunitarias calificadas. Tampoco confundas un motor de protocolos con un sistema completo de gestión de usuarios: puede necesitar ASP.NET Core Identity, un directorio o almacenamiento propio.

Externalizar login no externaliza tus decisiones de autorización, protección de recursos, secretos, auditoría ni respuesta ante incidentes.

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`. Instala `Microsoft.AspNetCore.Authentication.JwtBearer`. Usa `dotnet user-jwts` para probar localmente.

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("OrdersRead", policy =>
        policy.RequireAssertion(context =>
            HasScope(context.User, "orders.read")))
    .AddPolicy("AdminOnly", policy =>
        policy.RequireRole("admin"));

var app = builder.Build();

app.MapGet("/public", () => "public");

app.MapGet("/profile", (ClaimsPrincipal user) => new
    {
        Subject = user.FindFirst("sub")?.Value
            ?? user.FindFirst(ClaimTypes.NameIdentifier)?.Value
    })
    .RequireAuthorization();

app.MapGet("/orders", () => new[] { 42, 84 })
    .RequireAuthorization("OrdersRead");

app.MapGet("/admin", () => "admin")
    .RequireAuthorization("AdminOnly");

app.Run();

static bool HasScope(ClaimsPrincipal user, string expected) =>
    user.FindAll("scope")
        .SelectMany(claim => claim.Value.Split(
            ' ', StringSplitOptions.RemoveEmptyEntries))
        .Contains(expected, StringComparer.Ordinal);
```

Crea un token de desarrollo:

```text
dotnet user-jwts create --scope orders.read --role admin
```

Comportamiento:

```text
GET /orders sin token                            → 401
GET /orders con token autenticado sin scope      → 403
GET /orders con scope orders.read                → 200 [42,84]
GET /admin con role admin                        → 200 admin
```

El valor de `sub` y el token cambian en cada ejecución. No se muestran valores inventados.

-----

## Errores comunes

**1. Proteger todo solo con RequireAuthorization().**
Qué pasa: cualquier identidad válida accede a acciones sensibles.
Por qué: la policy predeterminada solo exige autenticación.
Arreglo: define policies por capacidad y recurso.

**2. Leer Identity.Name y asumir que siempre existe.**
Qué pasa: el saludo muestra null aunque el token sea válido.
Por qué: el proveedor no emitió el claim configurado como `NameClaimType`.
Arreglo: define el contrato de claims y configura el mapeo; usa `sub` como identificador estable, no el nombre visible.

**3. Usar RequireRole sin configurar RoleClaimType.**
Qué pasa: un usuario con roles aparenta no tenerlos.
Por qué: el claim real no coincide con el que espera el handler.
Arreglo: configura `TokenValidationParameters.RoleClaimType` según el proveedor.

**4. Comprobar scope como igualdad cuando contiene varios valores.**
Qué pasa: `orders.read orders.write` no coincide con `orders.read`.
Por qué: el claim usa una lista separada por espacios.
Arreglo: normaliza scopes en claims individuales o usa un requirement que los separe.

**5. Confiar en el ID del recurso enviado por el cliente.**
Qué pasa: un usuario cambia `/orders/42` por `/orders/43` y accede a datos ajenos.
Por qué: tener el scope no prueba propiedad.
Arreglo: autorización basada en recurso después de cargarlo.

**6. Presentar Auth0 o Duende como seguridad automática.**
Qué pasa: quedan audiences amplias, permisos excesivos o secretos expuestos.
Por qué: el proveedor emite identidad/tokens; tu sistema define y aplica acceso.
Arreglo: diseña amenazas, scopes, policies, rotación y monitoreo.

**7. Recomendar IdentityServer sin mencionar producto y licencia.**
Qué pasa: se elige una tecnología obsoleta o aparece un costo no evaluado.
Por qué: IdentityServer4 y Duende IdentityServer se confunden.
Arreglo: especifica Duende, su modelo de licencia y el costo de operación propia.

-----

## Según la versión de .NET

* **ASP.NET Core 5:** la autorización por policies, requirements y handlers ya era el modelo central.
* **ASP.NET Core 6 en adelante:** minimal APIs aplican policies con `RequireAuthorization`.
* **.NET 10:** `AddAuthorizationBuilder` permite registrar policies de forma fluida; Duende/Auth0 tienen ciclos y licencias independientes del runtime.

-----

## Cuándo sí y cuándo no

**Usa claims y policies cuando:** las reglas dependen de permisos emitidos, atributos confiables o requisitos reutilizables de la aplicación.

**No resuelvas todo con claims cuando:** la decisión depende del estado actual del recurso, propiedad, suscripción o datos revocables. Carga esos datos y usa autorización basada en recurso; no infles tokens con información cambiante.

-----

## Resumen en 5 líneas

1. Autenticación crea la identidad; autorización decide una acción sobre un recurso.
2. RequireAuthorization sin policy solo exige un usuario autenticado.
3. Policies expresan capacidades y pueden evaluar claims, scopes, roles o recursos.
4. Token inválido produce 401; permiso insuficiente produce 403.
5. Un proveedor centraliza identidad, pero tu API sigue siendo responsable de autorizar.

-----

## Para profundizar

<details>
<summary>Autorización basada en recurso</summary>

Un handler recibe el requisito y el objeto cargado. Puede comprobar que `order.OwnerId` coincide con `sub`, que el pedido no esté cerrado y que la acción tenga sentido. Devuelve éxito solo cuando todas las invariantes de acceso se cumplen. Así evitas codificar reglas complejas en el endpoint o confiar en IDs del cliente.

</details>

<details>
<summary>Fallback policy</summary>

Una fallback policy que exige autenticación protege endpoints olvidados. Los endpoints públicos deben marcarse explícitamente con `AllowAnonymous`. Reduce errores de omisión, pero no sustituye policies específicas para acciones sensibles.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los claims describen al usuario o cliente. Una policy agrupa requisitos, por ejemplo un scope o role, y se aplica con RequireAuthorization. Si el token es inválido se devuelve 401; si es válido pero no cumple la policy, 403.

### Respuesta ampliada (semi-senior)

Modelo policies por capacidades, no por pantallas ni un role Admin universal. Normalizo el contrato de claims del proveedor, especialmente scope y roles. Para objetos concretos uso autorización basada en recurso con IAuthorizationService. Distingo 401 de 403 y aplico fallback policy cuando conviene. Al elegir proveedor comparo operación, licencia, dependencia y soporte; externalizar autenticación no delega la autorización del dominio.

### Preguntas frecuentes de seguimiento

**1. ¿Un role es un permiso?**
No necesariamente. Un role agrupa responsabilidades; una policy puede traducirlo a capacidades junto con otros requisitos.

**2. ¿Por qué un token válido puede recibir 403?**
Porque autenticidad y vigencia no implican permiso para esa acción.

**3. ¿Auth0 reemplaza las policies de ASP.NET Core?**
No. Emite claims y tokens; la API debe aplicar sus reglas de acceso.

-----

## Práctica

**Ejercicio 1.** Un usuario tiene `orders.read` y solicita borrar un pedido ajeno. ¿Qué comprobaciones faltan?

<details>
<summary>Solución</summary>

Falta el permiso de borrado y la autorización basada en recurso. La API debe cargar el pedido y comprobar propiedad o una capacidad administrativa explícita.

</details>

**Ejercicio 2.** Una petición autenticada recibe 403. ¿Debes cambiarla automáticamente a 401?

<details>
<summary>Solución</summary>

No. 403 comunica que la identidad fue aceptada pero no cumple la policy. Convertirlo en 401 confunde al cliente y oculta errores de permisos.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Ejercicios de seguridad](Ejercicios.md) y vuelve al [índice del módulo](README.md).
