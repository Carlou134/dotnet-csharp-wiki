# Glosario: seguridad de APIs

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Access token:** credencial que un cliente presenta a una API; su formato puede ser JWT u opaco. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**Audience (`aud`):** destinatario para el que fue emitido un token. La API debe comprobar que es su audiencia. ([JWT bearer](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md))

**Authentication (autenticación):** proceso de establecer quién es un usuario o cliente. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**Authorization (autorización):** decisión de si una identidad puede ejecutar una acción sobre un recurso. ([Claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md))

**Authorization server:** servidor que autentica según el flujo aplicable, obtiene consentimiento cuando corresponde y emite tokens. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**Bearer token:** token que puede usar quien lo posee; por eso debe protegerse en tránsito y almacenamiento. ([JWT bearer](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md))

**Claim:** afirmación sobre el sujeto o cliente, como `sub`, `scope` o `role`. ([Claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md))

**Client:** aplicación que solicita autorización y usa un access token para llamar a una API. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**ID token:** JWT de OpenID Connect que informa al cliente sobre la autenticación del usuario; no se usa para acceder a una API. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**Issuer (`iss`):** autoridad que emitió el token. ([JWT bearer](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md))

**JSON Web Token (JWT):** formato compacto y firmado para transportar claims; su contenido codificado no está necesariamente cifrado. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**OAuth 2.0:** marco de autorización delegada: permite que un cliente obtenga un access token acotado para llamar a una API sin recibir la contraseña del usuario. No define autenticación. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**OpenID Connect (OIDC):** protocolo de autenticación construido sobre OAuth 2.0. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**PKCE:** protección del authorization code mediante un secreto efímero vinculado a la solicitud original. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**Policy (política):** conjunto nombrado de requisitos de autorización evaluados por ASP.NET Core. ([Claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md))

**Resource owner:** entidad capaz de conceder acceso a un recurso protegido; suele ser el usuario en flujos delegados. ([OAuth y OIDC](01-OAuth%202%20OpenID%20Connect%20y%20tokens.md))

**Resource server:** API que recibe y valida access tokens antes de autorizar el acceso. ([JWT bearer](02-Autenticacion%20JWT%20bearer%20en%20ASP.NET%20Core.md))

**Role:** pertenencia a una categoría funcional (`admin`, `support`). No equivale a un permiso concreto, y el nombre del claim depende del proveedor. ([Claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md))

**Scope:** permiso delegado solicitado y concedido para una API, como `orders.read`. ([Claims y políticas](03-Autorizacion%20con%20claims%20y%20politicas.md))
