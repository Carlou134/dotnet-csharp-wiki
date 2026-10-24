# Glosario: CQRS y mediator

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Aplanar:** convertir un dato anidado en una propiedad simple del DTO, como `Usuario.Rol.NombreRol` → `UsuarioDto.Rol`. ([DTOs](01-DTOs%20y%20mapeo.md))

**AutoMapper:** biblioteca que mapea objetos por convención de nombres usando reflexión; licencia comercial desde la versión 15. ([DTOs](01-DTOs%20y%20mapeo.md))

**Command:** mensaje que pide un cambio de estado y devuelve poco o nada. ([CQRS](02-CQRS%20y%20mediator%20con%20MediatR.md))

**Condición de carrera:** resultado que depende de qué operación concurrente llega primero, como dos registros con el mismo email que pasan juntos la validación. ([Validación](04-Validacion%20con%20pipeline%20behaviors.md))

**Contrato:** forma de los datos que un cliente espera recibir o enviar; cambiarla rompe a quien la consume. ([DTOs](01-DTOs%20y%20mapeo.md))

**CQRS (Command Query Responsibility Segregation):** separación de los caminos de escritura (commands) y de lectura (queries). ([CQRS](02-CQRS%20y%20mediator%20con%20MediatR.md))

**DTO (Data Transfer Object):** objeto sin comportamiento que define la forma de los datos que cruzan un límite. ([DTOs](01-DTOs%20y%20mapeo.md))

**Efecto secundario:** trabajo que ocurre como consecuencia de un caso de uso sin ser su objetivo, como enviar un email o invalidar una caché. ([Notificaciones](03-Notificaciones%20en%20proceso%20con%20INotification.md))

**FluentValidation:** biblioteca para declarar reglas de validación con código. ([Validación](04-Validacion%20con%20pipeline%20behaviors.md))

**Handler:** clase que ejecuta un command, una query o una notificación concreta. ([CQRS](02-CQRS%20y%20mediator%20con%20MediatR.md))

**IExceptionHandler:** interfaz de ASP.NET Core 8+ para convertir excepciones en respuestas HTTP desde el middleware de excepciones. ([Validación](04-Validacion%20con%20pipeline%20behaviors.md))

**Mapeo:** copia de datos de un tipo a otro, como de una entidad a su DTO. ([DTOs](01-DTOs%20y%20mapeo.md))

**Notificación:** mensaje que comunica un hecho pasado y puede tener cero o muchos handlers. ([Notificaciones](03-Notificaciones%20en%20proceso%20con%20INotification.md))

**Outbox:** tabla donde se guarda un evento en la misma transacción que el cambio, para entregarlo después de forma confiable. ([Notificaciones](03-Notificaciones%20en%20proceso%20con%20INotification.md))

**Patrón mediator:** objeto intermedio que recibe mensajes y los entrega a quien los procesa, sin que el emisor conozca al receptor. ([CQRS](02-CQRS%20y%20mediator%20con%20MediatR.md))

**Pipeline behavior:** clase `IPipelineBehavior<TRequest, TResponse>` que se ejecuta alrededor de cada handler de MediatR, como un middleware. ([Validación](04-Validacion%20con%20pipeline%20behaviors.md))

**Publicador de notificaciones (notification publisher):** estrategia que decide cómo se ejecutan los handlers de una notificación: en secuencia, en paralelo o aislando errores. ([Notificaciones](03-Notificaciones%20en%20proceso%20con%20INotification.md))

**Query:** mensaje que pide datos sin modificar nada. ([CQRS](02-CQRS%20y%20mediator%20con%20MediatR.md))

**Validator:** clase `AbstractValidator<T>` con las reglas de validación de un tipo. ([Validación](04-Validacion%20con%20pipeline%20behaviors.md))

**Vertical slice:** organización del código por caso de uso en lugar de por capa técnica. ([CQRS](02-CQRS%20y%20mediator%20con%20MediatR.md))
