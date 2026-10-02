# Glosario: diseño de APIs REST

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**API (*Application Programming Interface*):** contrato que permite a un programa usar las funciones de otro. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**`[ApiController]`:** atributo de ASP.NET Core que activa la validación automática del modelo, la inferencia de orígenes de parámetros y los errores en formato ProblemDetails. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**Cambio que rompe (*breaking change*):** cambio que hace fallar a un cliente que funcionaba. ([Versionamiento](03-Versionamiento%20de%20APIs.md))

**Código de estado:** número de 3 dígitos de la respuesta HTTP que indica el resultado. 2xx éxito, 4xx error del cliente, 5xx error del servidor. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**Contrato:** todo lo que un cliente asume de una API: rutas, campos, tipos, códigos y comportamiento. ([Versionamiento](03-Versionamiento%20de%20APIs.md))

**Deprecación:** anunciar que una versión dejará de existir, sin quitarla todavía. ([Versionamiento](03-Versionamiento%20de%20APIs.md))

**DTO (*Data Transfer Object*):** tipo que define la forma de los datos que entran o salen de la API. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**Endpoint:** combinación de método HTTP y ruta (`GET /api/pedidos/5`). ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**Enlace (*link*):** objeto con la URL (`href`), la relación (`rel`) y, a menudo, el método HTTP. ([HATEOAS](04-HATEOAS.md))

**HAL / JSON:API / Siren:** formatos estándar para representar enlaces e hipermedia en JSON. ([HATEOAS](04-HATEOAS.md))

**HATEOAS (*Hypermedia As The Engine Of Application State*):** las respuestas incluyen enlaces a las acciones disponibles según el estado del recurso. ([HATEOAS](04-HATEOAS.md))

**Hipermedia:** contenido que incluye enlaces a otros recursos o acciones. ([HATEOAS](04-HATEOAS.md))

**Idempotente:** repetir la misma petición una o varias veces deja el servidor en el mismo estado (GET, PUT, DELETE). ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**Idempotency key:** identificador único que envía el cliente para que reintentar un POST no lo ejecute dos veces. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**`IExceptionHandler`:** interfaz de .NET 8+ para convertir excepciones no controladas en respuestas HTTP de forma centralizada. ([ProblemDetails](02-Errores%20con%20ProblemDetails.md))

**Lector tolerante (*tolerant reader*):** cliente que ignora los campos que no conoce en lugar de fallar. ([Versionamiento](03-Versionamiento%20de%20APIs.md))

**`LinkGenerator`:** servicio que genera URLs a partir de acciones y valores de ruta, también fuera de los controladores. ([HATEOAS](04-HATEOAS.md))

**Manejo centralizado de errores:** un único lugar que convierte las excepciones en respuestas HTTP. ([ProblemDetails](02-Errores%20con%20ProblemDetails.md))

**Middleware:** componente que procesa cada petición y respuesta en la cadena (*pipeline*) de ASP.NET Core. ([ProblemDetails](02-Errores%20con%20ProblemDetails.md))

**Modelo de madurez de Richardson:** clasificación de APIs en niveles 0 a 3; el nivel 3 usa hipermedia. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**ProblemDetails:** objeto JSON estándar para describir un error HTTP (`type`, `title`, `status`, `detail`, `instance`). Tipo de contenido `application/problem+json`. ([ProblemDetails](02-Errores%20con%20ProblemDetails.md))

**Recurso:** cualquier "cosa" que la API expone y que tiene una URI. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**`rel` (relación):** nombre que indica qué significa un enlace (`self`, `next`, `cancelar`). ([HATEOAS](04-HATEOAS.md))

**REST (*Representational State Transfer*):** estilo de arquitectura para APIs sobre HTTP basado en recursos. ([Principios REST](01-Principios%20REST%20y%20codigos%20de%20estado.md))

**RFC 9457:** estándar de ProblemDetails (2023); reemplaza al RFC 7807 (2016). ([ProblemDetails](02-Errores%20con%20ProblemDetails.md))

**Sunset:** fecha en que una versión deprecada se apaga; también un header HTTP (RFC 8594). ([Versionamiento](03-Versionamiento%20de%20APIs.md))

**Versionamiento:** publicar los cambios que rompen como una versión nueva mientras la anterior sigue funcionando. ([Versionamiento](03-Versionamiento%20de%20APIs.md))
