# Glosario: eventos y Publish/Subscribe

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Acoplamiento:** cuánto necesita saber un componente de otro para funcionar. Puede ser de conocimiento, temporal o de contrato. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Acuse de recibo (*ack*):** aviso del consumidor al broker de que terminó con un mensaje; sin él, el broker lo reentrega. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Aislamiento de errores:** el fallo de un suscriptor no impide que los demás reciban el evento. ([Pub/Sub en memoria](02-Publish%20Subscribe%20en%20memoria.md))

**Al menos una vez (*at-least-once*):** garantía de entrega sin pérdidas, pero con posibles duplicados. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Broker de mensajes:** servidor que recibe, guarda y entrega mensajes (RabbitMQ, Azure Service Bus, Kafka). ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Cola de mensajes fallidos (*dead-letter queue*, DLQ):** lugar donde terminan los mensajes que no se pudieron procesar. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Cola (*queue*) y tópico (*topic*):** una cola entrega cada mensaje a un consumidor; un tópico, a cada suscripción. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Comando:** mensaje que pide una acción a un destinatario concreto; se nombra en imperativo y puede rechazarse. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Compensación:** acción que deshace el efecto de un paso anterior cuando un paso posterior falla (liberar el stock si el pago falla). ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Consistencia eventual:** las distintas partes del sistema terminan siendo coherentes, pero no en el mismo instante. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Consumidor (*subscriber*, *handler*):** componente que reacciona a un evento. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Copia al escribir (*copy-on-write*):** en lugar de modificar una colección compartida, se reemplaza por una nueva. ([Pub/Sub en memoria](02-Publish%20Subscribe%20en%20memoria.md))

**Coreografía y orquestación:** formas de coordinar procesos entre servicios: cada uno reacciona a eventos (coreografía) o un coordinador envía comandos (orquestación). ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Doble escritura:** escribir en dos sistemas (base de datos y broker) sin una transacción común. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**EDA (*Event-Driven Architecture*):** arquitectura en la que los componentes se comunican publicando y reaccionando a eventos. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Evento:** mensaje inmutable que describe un hecho que ya ocurrió; se nombra en pasado. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Evento de dominio / de integración:** el de dominio ocurre y se maneja dentro de un servicio; el de integración cruza servicios y es un contrato público. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Fuga de memoria por suscripción:** un objeto que nunca se desuscribe queda referenciado por el bus y no se libera. ([Pub/Sub en memoria](02-Publish%20Subscribe%20en%20memoria.md))

**Idempotente:** procesar el mismo mensaje una o varias veces deja el mismo resultado. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Outbox (bandeja de salida):** tabla donde se guardan los eventos en la misma transacción que el cambio de datos, para publicarlos después. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Productor (*publisher*):** componente que publica un evento. ([Eventos y EDA](01-Eventos%20y%20arquitectura%20dirigida%20por%20eventos.md))

**Publish/Subscribe (Pub/Sub):** patrón en el que publicadores y suscriptores se comunican a través de un intermediario sin conocerse. ([Pub/Sub en memoria](02-Publish%20Subscribe%20en%20memoria.md))

**Saga:** proceso de negocio que cruza varios servicios y usa compensaciones en lugar de una transacción distribuida. ([Entrega confiable](03-Entrega%20confiable%20e%20idempotencia.md))

**Suscripción:** registro de un handler para un tipo de evento; debe poder cancelarse (`IDisposable`). ([Pub/Sub en memoria](02-Publish%20Subscribe%20en%20memoria.md))
