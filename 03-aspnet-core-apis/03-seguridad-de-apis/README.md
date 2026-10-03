# Seguridad de APIs con OAuth 2.0 y OpenID Connect

En esta carpeta aprendes a separar autenticación, delegación y autorización: OAuth 2.0 concede acceso, OpenID Connect autentica al usuario para el cliente, y una API de ASP.NET Core valida access tokens y aplica políticas sobre recursos concretos.

-----

## Antes de empezar

Conviene que ya tengas:

* Diseño de recursos y códigos `401`/`403`: [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md).
* Endpoints, grupos y `RequireAuthorization`: [Minimal APIs](../02-minimal-apis/README.md).
* Configuración e inyección de dependencias de ASP.NET Core.

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. OAuth 2.0, OpenID Connect y tokens](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md) | Actores, autorización frente a autenticación, flujos, access token frente a ID token y JWT | HTTP, APIs REST |
| [2. Autenticación JWT bearer en ASP.NET Core](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md) | Validación de firma, issuer, audience y expiración; `AddJwtBearer`; `401`; pruebas con `dotnet user-jwts` | OAuth/OIDC, minimal APIs |
| [3. Autorización con claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md) | Scopes, roles, policies, `401` frente a `403`, recursos y elección de proveedor | JWT bearer |

Para practicar: [Ejercicios de seguridad](Ejercicios.md), con los cinco ejercicios y los tres retos de la Sesión 14 corregidos.

-----

## El mapa completo en una mirada

```text
OAuth 2.0          delega AUTORIZACIÓN                 → access token para una API
OpenID Connect     autentica al usuario para el CLIENTE → ID token para el cliente
JWT                formato firmado, no protocolo        → puede contener access o ID token

Cliente ── obtiene access token ──> Authorization Server
Cliente ── Bearer access_token ──> Resource Server (API)
API     ── valida firma + issuer + audience + exp ──> identidad autenticada
API     ── evalúa scope/claim/policy + recurso ─────> autorización

Token ausente o inválido → 401
Token válido sin permiso → 403
ID token enviado a la API → token equivocado
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Los ejemplos usan minimal APIs de .NET 10. Para pruebas locales se usa `dotnet user-jwts`; en producción los tokens deben proceder de un authorization server confiable.

-----

## Después de esta carpeta

La validación de tokens es solo una capa. Continúa con gestión segura de secretos y claves, protección de datos, CORS y CSRF según el tipo de cliente, OWASP API Security Top 10, *rate limiting*, auditoría y pruebas de autorización por recurso.
