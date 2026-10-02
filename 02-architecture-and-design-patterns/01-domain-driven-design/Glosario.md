# Glosario: Domain-Driven Design

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Agregado (*aggregate*):** grupo de entidades y value objects que se modifica, valida y guarda como una unidad. ([Agregados](04-Agregados.md))

**Auto-validación:** el objeto valida sus datos al crearse, de modo que si existe, es válido. ([Value objects](03-Value%20objects.md))

**Bloques tácticos:** los patrones de código de DDD: entidades, value objects, agregados, servicios de dominio, eventos de dominio y repositorios. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Bounded context (contexto delimitado):** parte de un sistema con su propio modelo y su propio lenguaje; la misma palabra puede significar cosas distintas en contextos distintos. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Ciclo de vida:** los estados por los que pasa una entidad desde que se crea hasta que se archiva o elimina. ([Entidades](02-Entidades.md))

**Consistencia eventual:** los cambios entre agregados distintos se completan poco después, en otra transacción, normalmente mediante eventos de dominio. ([Agregados](04-Agregados.md))

**Consistencia transaccional:** todo lo que está dentro de un agregado se guarda en la misma transacción: o todo, o nada. ([Agregados](04-Agregados.md))

**DDD (*Domain-Driven Design*):** enfoque de diseño que pone el modelo del negocio en el centro del software. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Dominio:** el área del negocio que resuelve el software. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Entidad:** objeto con identidad propia que perdura y cambia a lo largo del tiempo; se compara por su Id. ([Entidades](02-Entidades.md))

**Evento de dominio:** algo relevante que ocurrió en el dominio (`PedidoConfirmado`), publicado para que otras partes reaccionen. ([Agregados](04-Agregados.md))

**Excepción de dominio:** excepción propia (por ejemplo, `DomainException`) que indica que se violó una regla del negocio. ([Agregados](04-Agregados.md))

**Frontera del agregado:** el límite que separa los objetos protegidos por la raíz de los que quedan afuera. ([Agregados](04-Agregados.md))

**Identidad:** el dato que distingue a una entidad de todas las demás, normalmente un `Id` inmutable. ([Entidades](02-Entidades.md))

**Igualdad estructural (por valor):** dos objetos son iguales si todos sus valores son iguales. Es la de los records. ([Value objects](03-Value%20objects.md))

**Igualdad por identidad:** dos objetos representan la misma entidad si tienen el mismo `Id` (y tipo). ([Entidades](02-Entidades.md))

**Inmutable:** que no cambia después de crearse; para "modificarlo" se crea otro. ([Value objects](03-Value%20objects.md))

**Invariante:** regla que el modelo debe cumplir siempre. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Lenguaje ubicuo (*ubiquitous language*):** vocabulario común entre el negocio y los desarrolladores, usado tal cual en el código. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Método de fábrica (*factory*):** método estático que crea un objeto válido y expresa la intención (`Cliente.Registrar`, `Pedido.Crear`). ([Entidades](02-Entidades.md))

**Modelo anémico:** clases con solo propiedades públicas, sin comportamiento; la lógica queda dispersa en servicios. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Modelo rico (de dominio):** clases que combinan datos y comportamiento y protegen sus invariantes. ([Modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md))

**Obsesión por los primitivos (*primitive obsession*):** usar `string`, `int` o `decimal` para conceptos del negocio que tienen reglas propias. ([Value objects](03-Value%20objects.md))

**Raíz del agregado (*aggregate root*):** la entidad principal del agregado y la única a la que se accede desde afuera. ([Agregados](04-Agregados.md))

**Repositorio:** abstracción que guarda y recupera agregados completos; hay uno por agregado, no por tabla. ([Agregados](04-Agregados.md))

**Servicio de dominio:** clase sin estado con lógica del negocio que no pertenece a una sola entidad o agregado. ([Agregados](04-Agregados.md))

**Value object (objeto de valor):** objeto sin identidad, inmutable, definido y comparado por sus valores. ([Value objects](03-Value%20objects.md))
