# OAuth 2.0, OpenID Connect y tokens

## En una frase

OAuth 2.0 delega acceso a una API; OpenID Connect añade autenticación para que el cliente conozca al usuario; JWT es solo uno de los formatos posibles para representar tokens.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo se modelan recursos y métodos HTTP: [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md).
* Qué es un cliente y una API: [Endpoints y grupos](../02-minimal-apis/01-Endpoints%20y%20grupos%20de%20rutas.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **OAuth 2.0:** marco de autorización delegada.
* **OpenID Connect:** protocolo de autenticación sobre OAuth 2.0.
* **Access token:** credencial dirigida a una API.
* **ID token:** resultado de autenticación dirigido al cliente.
* **JWT:** formato compacto firmado que transporta claims.
* **PKCE:** protección del authorization code frente a interceptación.

-----

## El problema

Una aplicación necesita consultar los pedidos de una persona. El diseño ingenuo pide su contraseña y la reenvía a cada servicio:

```text
usuario ── contraseña ──> aplicación ── contraseña ──> pedidos
                                      └─ contraseña ──> pagos
```

Ahora cada aplicación almacena credenciales, puede impersonar al usuario sin límites y obliga a cambiar todas las integraciones cuando la contraseña rota. Tampoco existe una concesión acotada a “leer pedidos”.

OAuth separa las responsabilidades. Pero decir “OAuth autentica al usuario” también es incorrecto: OAuth define delegación de acceso. OpenID Connect añade la capa que permite al cliente verificar la autenticación del usuario.

-----

## Cómo funciona

### 1. Los cuatro roles de OAuth 2.0

| Rol | Responsabilidad |
| --- | --- |
| Resource owner | Puede conceder acceso al recurso; suele ser el usuario |
| Client | Solicita y usa la autorización |
| Authorization server | Emite tokens según el flujo y la concesión |
| Resource server | API que valida el access token y protege recursos |

El client no entrega la contraseña del usuario a la API. Obtiene un access token con audiencia y permisos acotados.

### 2. OpenID Connect responde una pregunta distinta

```text
OAuth 2.0:       ¿puede este cliente llamar a Orders API con orders.read?
OpenID Connect:  ¿qué usuario se autenticó ante este cliente?
```

OIDC introduce el scope `openid`, el ID token, discovery y reglas de validación de identidad. Una API normalmente valida el **access token**. El **ID token pertenece al cliente** y no debe presentarse como autorización ante la API.

### 3. Access token, ID token y refresh token no son intercambiables

| Token | Consumidor | Uso |
| --- | --- | --- |
| Access token | Resource server | Autorizar llamadas a una API |
| ID token | Client OIDC | Verificar el resultado del login y obtener identidad básica |
| Refresh token | Authorization server | Solicitar nuevos tokens según la concesión |

Un access token puede ser JWT u opaco. El cliente debe tratarlo como una cadena; interpretar sus claims acopla el cliente a un contrato que pertenece a la API y al emisor.

### 4. JWT no significa secreto ni autorización

Un JWT firmado ofrece integridad y autenticidad cuando se valida correctamente. Su payload usa codificación base64url, no cifrado: cualquiera que posea el token puede leer sus claims.

```text
header.payload.signature
```

La firma válida prueba quién lo emitió y que no fue alterado. La API todavía debe validar `iss`, `aud`, expiración y permisos. Un token válido para otra API sigue siendo inválido para esta.

### 5. Elige el flujo según el cliente

* **Authorization Code + PKCE:** aplicaciones con usuario, incluidas aplicaciones públicas. El navegador recibe un código breve; el cliente demuestra el `code_verifier` al canjearlo.
* **Client Credentials:** comunicación máquina a máquina, sin usuario. Los permisos representan a la aplicación.
* **Device Authorization:** dispositivos con entrada limitada.

No uses *implicit flow* como diseño nuevo ni *resource owner password credentials*. Las buenas prácticas actuales exigen no recopilar la contraseña del usuario en el client y proteger el authorization code con PKCE.

### 6. Bearer significa “quien lo posee puede usarlo”

El access token viaja normalmente así:

```http
Authorization: Bearer access-token
```

HTTPS es obligatorio. Evita registrarlo, colocarlo en URLs o exponerlo a scripts innecesarios. Para riesgos mayores existen tokens ligados al emisor mediante DPoP o mTLS.

-----

## Ejemplo completo

Este programa de consola no emite tokens reales: modela un conjunto recibido desde un proveedor para demostrar el destino correcto de cada token.

```csharp
using System.Net.Http.Headers;

var tokens = new TokenSet(
    IdToken: "id-token-for-web-client",
    AccessToken: "access-token-for-orders-api");

Console.WriteLine("El cliente valida el ID token para completar el login.");

using var request = new HttpRequestMessage(
    HttpMethod.Get,
    "https://orders.example.com/orders");

request.Headers.Authorization =
    new AuthenticationHeaderValue("Bearer", tokens.AccessToken);

Console.WriteLine($"Token enviado a la API: {tokens.AccessToken}");
Console.WriteLine($"ID token enviado a la API: {ReferenceEquals(tokens.IdToken, tokens.AccessToken)}");

public sealed record TokenSet(string IdToken, string AccessToken);
```

Salida:

```text
El cliente valida el ID token para completar el login.
Token enviado a la API: access-token-for-orders-api
ID token enviado a la API: False
```

En una aplicación real una librería OIDC obtiene y valida los tokens. No construyas ni parses el flujo manualmente con peticiones improvisadas.

-----

## Errores comunes

**1. Decir que OAuth autentica al usuario.**
Qué pasa: se usa un protocolo de autorización como prueba de identidad.
Por qué: OAuth no define cómo autenticar al usuario ni un ID token.
Arreglo: usa OpenID Connect para login.

**2. Enviar el ID token a la API.**
Qué pasa: la API recibe un token cuya audiencia es el cliente.
Por qué: se confunden identidad del cliente y autorización del recurso.
Arreglo: envía el access token destinado a esa API.

**3. Suponer que todo access token es JWT.**
Qué pasa: el cliente intenta decodificar un token opaco o depende de claims internos.
Por qué: OAuth no fija un único formato.
Arreglo: el cliente trata el token como cadena; la API aplica su método de validación.

**4. Creer que un JWT está cifrado.**
Qué pasa: se incluyen secretos o información sensible en claims legibles.
Por qué: base64url solo codifica.
Arreglo: minimiza claims; usa cifrado solo cuando el diseño realmente lo requiera.

**5. Usar password grant o implicit flow en un diseño nuevo.**
Qué pasa: se exponen credenciales o tokens a superficies innecesarias.
Por qué: son patrones anteriores a las recomendaciones modernas.
Arreglo: Authorization Code + PKCE o un flujo apropiado al cliente.

**6. Llamar “desacoplado y seguro” al sistema solo por usar tokens.**
Qué pasa: se ignoran rotación, revocación, almacenamiento, scopes y compromiso del proveedor.
Por qué: delegar identidad cambia el riesgo; no lo elimina.
Arreglo: modela amenazas y responsabilidades completas.

-----

## Según la versión de .NET

* **Los protocolos no dependen de .NET:** OAuth 2.0 y OpenID Connect tienen especificaciones propias.
* **ASP.NET Core moderno:** ofrece handlers para OIDC en clientes web y JWT bearer en APIs.
* **Buenas prácticas actuales:** Authorization Code + PKCE reemplaza diseños nuevos basados en implicit y password grant; .NET 10 no cambia esa separación conceptual.

-----

## Cuándo sí y cuándo no

**Usa OAuth/OIDC cuando:** necesitas login federado, SSO, acceso delegado o autorización máquina a máquina con un emisor central confiable.

**No agregues un proveedor externo por reflejo cuando:** una aplicación aislada y de bajo riesgo puede resolverse con autenticación integrada más simple. Centralizar identidad añade dependencia, configuración, costo y un componente crítico que debes operar o contratar.

-----

## Resumen en 5 líneas

1. OAuth 2.0 delega autorización; OpenID Connect autentica al usuario para el cliente.
2. El access token se presenta a la API y el ID token pertenece al cliente.
3. Un access token puede ser JWT u opaco; JWT es un formato, no un protocolo.
4. Una firma válida no reemplaza validar issuer, audience, expiración y permisos.
5. Usa Authorization Code + PKCE para usuarios y Client Credentials para máquina a máquina.

-----

## Para profundizar

<details>
<summary>Authorization Code + PKCE</summary>

El cliente genera un `code_verifier` aleatorio y envía su transformación como `code_challenge`. Al canjear el código presenta el verifier. Un atacante que intercepte solo el authorization code no puede usarlo sin ese secreto efímero. PKCE no reemplaza validar `state`, redirect URIs y, en OIDC, `nonce`.

</details>

<details>
<summary>Bearer frente a sender-constrained</summary>

Un bearer token robado puede reutilizarse mientras sea válido. DPoP y mTLS vinculan el token a una clave o certificado que el cliente debe demostrar poseer. Añaden seguridad, pero también complejidad de claves, proxies y compatibilidad.

</details>

-----

## En entrevista

### Respuesta corta (junior)

OAuth 2.0 permite que una aplicación acceda a una API con permisos delegados sin recibir la contraseña del usuario. OpenID Connect añade autenticación y entrega un ID token al cliente. La API recibe un access token, que puede ser JWT u opaco.

### Respuesta ampliada (semi-senior)

Separo authorization server, client y resource server. Para usuarios elijo Authorization Code + PKCE; para máquina a máquina, Client Credentials. El cliente valida el ID token para su sesión y trata el access token como opaco. La API valida que el access token fue emitido por la autoridad esperada, está destinado a su audience, no expiró y contiene permisos suficientes. Considero almacenamiento, revocación, rotación y tokens ligados al emisor según el riesgo.

### Preguntas frecuentes de seguimiento

**1. ¿OAuth es autenticación?**
No. OpenID Connect añade autenticación sobre OAuth 2.0.

**2. ¿Puede enviarse el ID token a la API?**
No. Su audiencia es el cliente; la API espera un access token.

**3. ¿Todos los JWT están cifrados?**
No. Lo habitual es que estén firmados y su payload sea legible.

-----

## Práctica

**Ejercicio 1.** Elige el token correcto: mostrar el nombre en la sesión del cliente, llamar a Orders API y renovar una concesión.

<details>
<summary>Solución</summary>

ID token para completar la autenticación en el cliente, access token para Orders API y refresh token ante el authorization server.

</details>

**Ejercicio 2.** ¿Qué flujo usarías para un proceso nocturno sin usuario que llama a una API interna?

<details>
<summary>Solución</summary>

Client Credentials. El token representa al proceso cliente, no a un usuario; sus permisos deben ser los mínimos necesarios.

</details>

-----

## Siguiente lección

[Autenticación JWT bearer en ASP.NET Core](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md)
