# Autenticación JWT bearer en ASP.NET Core

## En una frase

El handler JWT bearer convierte un access token confiable en un `ClaimsPrincipal` solo después de validar firma, emisor, audiencia y vigencia; leer el payload no autentica a nadie.

-----

## Antes de empezar

Conviene que ya sepas:

* Diferenciar access token, ID token y JWT: [OAuth 2.0, OpenID Connect y tokens](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md).
* Proteger endpoints con `RequireAuthorization`: [Minimal APIs](../02-minimal-apis/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Issuer (`iss`):** autoridad que emitió el token.
* **Audience (`aud`):** destinatario autorizado del token.
* **Bearer token:** credencial utilizable por quien la posee.
* **Resource server:** API que valida el access token.

-----

## El problema

Este código no autentica:

```csharp
var token = request.Headers.Authorization.ToString()["Bearer ".Length..];
var payload = token.Split('.')[1];
// decodificar JSON y confiar en role = admin
```

Cualquiera puede fabricar header y payload. La seguridad de un JWT no está en que “parezca un token”, sino en verificar criptográficamente la firma y comprobar que sus restricciones corresponden a esta API.

Tampoco alcanza con comprobar que existe el header `Authorization`. Eso solo demuestra que el cliente envió texto.

-----

## Cómo funciona

### 1. La API registra un esquema y un handler

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Authentication:Authority"];
        options.Audience = builder.Configuration["Authentication:Audience"];
        options.MapInboundClaims = false;
    });

builder.Services.AddAuthorization();
```

`Authority` permite descubrir issuer y claves públicas mediante los metadatos del proveedor. `Audience` identifica esta API. `MapInboundClaims = false` conserva los nombres originales como `sub`, `scope` y `roles` en lugar de transformarlos a nombres históricos de .NET.

No escribas dominio, audience ni secretos directamente en el código. Authority y audience no son secretos, pero cambian por entorno; claves privadas y client secrets sí deben almacenarse en un sistema de secretos.

### 2. La validación es un conjunto, no una firma aislada

La API debe comprobar:

| Validación | Pregunta |
| --- | --- |
| Firma | ¿Lo emitió una autoridad confiable y no fue alterado? |
| Issuer | ¿Es exactamente el emisor configurado? |
| Audience | ¿Fue emitido para esta API? |
| Expiración y vigencia | ¿Puede usarse ahora? |
| Tipo y claims esperados | ¿Es un access token con el contrato correcto? |

Un token firmado por la misma autoridad pero destinado a otra API debe rechazarse.

### 3. Authentication crea la identidad; authorization decide el acceso

```csharp
app.MapGet("/private", () => "Protegido")
   .RequireAuthorization();
```

Sin credencial o con token inválido, el handler produce un challenge y la respuesta es `401 Unauthorized`. Con token válido se crea `HttpContext.User`; todavía falta evaluar las políticas del endpoint.

### 4. El orden del middleware puede ser explícito

En minimal APIs, `WebApplication` registra automáticamente autenticación y autorización cuando agregas sus servicios. Si necesitas controlar el orden respecto de CORS u otro middleware:

```csharp
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
```

Autenticación debe ocurrir antes de autorización. No agregues middleware duplicado “por seguridad”.

### 5. Las claves rotan mediante discovery

Con un proveedor estándar, el handler obtiene metadatos y claves públicas. El emisor puede publicar una nueva clave con otro `kid`; el middleware actualiza su configuración. No copies una clave pública permanentemente al código salvo que controles explícitamente su distribución y rotación.

### 6. El token es una credencial sensible

Usa HTTPS, expiraciones cortas y permisos mínimos. No lo escribas en logs, query strings ni mensajes de error. Un JWT firmado no puede “desemitirse”; la revocación inmediata requiere tokens de corta vida, introspection para tokens de referencia u otra estrategia del proveedor.

-----

## Ejemplo completo

`Program.cs` de `dotnet new web`. Instala `Microsoft.AspNetCore.Authentication.JwtBearer`. Para desarrollo, `dotnet user-jwts` genera un token y configura validación local sin depender de un proveedor público.

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();

builder.Services.AddAuthorization();

var app = builder.Build();

app.MapGet("/public", () => TypedResults.Ok(new { Message = "public" }));

app.MapGet("/private", () =>
        TypedResults.Ok(new { Message = "authenticated" }))
    .RequireAuthorization();

app.Run();
```

En la carpeta del proyecto:

```text
dotnet user-jwts create
```

Comportamiento:

```text
GET /public                                      → 200 {"message":"public"}
GET /private                                     → 401
GET /private  Authorization: Bearer <token válido> → 200 {"message":"authenticated"}
```

El token generado y su identificador cambian en cada ejecución, por eso no se inventan aquí. `dotnet user-jwts` es para desarrollo, no un emisor de producción.

-----

## Errores comunes

**1. Decodificar el JWT y confiar en sus claims.**
Qué pasa: un atacante modifica `role` o `sub` y la API lo acepta.
Por qué: decodificar no verifica la firma.
Arreglo: usa `AddJwtBearer` con validación completa.

**2. Validar la firma pero no audience.**
Qué pasa: se acepta un token legítimo emitido para otra API.
Por qué: se confunde confianza en el emisor con permiso para este recurso.
Arreglo: configura y valida la audiencia exacta.

**3. Usar un ID token como bearer de la API.**
Qué pasa: se acepta un token destinado al cliente OIDC.
Por qué: se confunden login y acceso a recursos.
Arreglo: la API solo acepta access tokens destinados a ella.

**4. Probar producción contra un servidor demo público.**
Qué pasa: disponibilidad, configuración y tokens quedan fuera de tu control.
Por qué: un ejemplo didáctico se convirtió en dependencia.
Arreglo: usa `dotnet user-jwts` localmente y un proveedor propio contratado o administrado en producción.

**5. Desactivar HTTPS para metadatos en producción.**
Qué pasa: se exponen discovery y claves a manipulación en tránsito.
Por qué: se conserva una excepción de desarrollo.
Arreglo: mantén `RequireHttpsMetadata = true`, que es el valor seguro predeterminado.

**6. Registrar el token para depurar.**
Qué pasa: una credencial reutilizable termina en logs y copias de seguridad.
Por qué: se trata el header como texto inocuo.
Arreglo: registra motivo y metadatos mínimos, nunca el bearer completo.

-----

## Según la versión de .NET

* **ASP.NET Core 6 en adelante:** minimal APIs admiten `RequireAuthorization` y autenticación JWT bearer.
* **ASP.NET Core moderno:** puede cargar esquemas desde `Authentication:Schemes` y agrega automáticamente los middleware cuando no necesitas ordenar el pipeline manualmente.
* **.NET 10:** `Microsoft.AspNetCore.Authentication.JwtBearer` sigue siendo el handler estándar para validar access tokens JWT en APIs.

-----

## Cuándo sí y cuándo no

**Usa JWT bearer cuando:** tu authorization server emite access tokens JWT para esta API y necesitas validación local mediante firma y claims.

**No lo uses cuando:** el proveedor emite tokens opacos que requieren introspection, necesitas revocación inmediata incompatible con validación local o estás construyendo una aplicación web con sesión y cookies donde el bearer no aporta valor.

-----

## Resumen en 5 líneas

1. Un JWT se autentica verificando firma, issuer, audience y vigencia.
2. Authority descubre metadatos y claves; audience identifica la API destinataria.
3. AddJwtBearer crea el ClaimsPrincipal solo después de validar el access token.
4. Token ausente o inválido produce 401; los permisos se evalúan después.
5. dotnet user-jwts sirve para desarrollo, no para emitir tokens de producción.

-----

## Para profundizar

<details>
<summary>Clock skew y expiración</summary>

Los servidores pueden diferir algunos segundos. Los validadores admiten una tolerancia de reloj, pero aumentarla demasiado prolonga tokens vencidos. Sincroniza relojes y usa una tolerancia pequeña acorde con tu infraestructura.

</details>

<details>
<summary>Múltiples emisores</summary>

Una API puede registrar esquemas separados o un `PolicyScheme` que seleccione el handler. No elijas el emisor leyendo claims sin validarlos y aceptándolos directamente: el valor solo sirve para enrutar hacia un conjunto de validación previamente confiable.

</details>

-----

## En entrevista

### Respuesta corta (junior)

JWT bearer valida el access token recibido en `Authorization`. Comprueba firma, issuer, audience y expiración; si es válido crea el usuario con sus claims. `RequireAuthorization` exige esa identidad en el endpoint.

### Respuesta ampliada (semi-senior)

Configuro un esquema bearer con una authority confiable y la audience exacta de la API. Dejo que discovery y JWKS administren claves y rotación, con HTTPS obligatorio. Conservo claims sin mapeo cuando necesito nombres estándar y no registro tokens. Distingo autenticación de autorización: una firma válida solo crea un principal; policies y handlers deciden si puede actuar sobre el recurso.

### Preguntas frecuentes de seguimiento

**1. ¿Leer el payload valida el JWT?**
No. Solo decodifica datos controlables por el atacante.

**2. ¿Qué diferencia hay entre 401 y 403?**
401 indica credencial ausente o inválida; 403, identidad válida sin autorización suficiente.

**3. ¿Authority y audience son secretos?**
No, pero son configuración por entorno. Client secrets y claves privadas sí son secretos.

-----

## Práctica

**Ejercicio 1.** Una API de inventario acepta un token válido emitido para pagos. ¿Qué validación falta?

<details>
<summary>Solución</summary>

La audience. Inventario debe exigir su propio identificador en `aud`; compartir issuer no vuelve intercambiables los tokens.

</details>

**Ejercicio 2.** ¿Por qué no debes mostrar el token completo al explicar un error `401`?

<details>
<summary>Solución</summary>

Porque es una credencial bearer. Quien accede al log podría reutilizarla hasta que venza. Registra una categoría de error y correlación, no el token.

</details>

-----

## Siguiente lección

[Autorización con claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md)
