# Glosario: Clean y Hexagonal Architecture

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Actor primario y secundario:** quien inicia una interacción con la aplicación (un usuario, un job) frente a quien la aplicación usa (una base, un proveedor). ([Puertos y adaptadores](02-Puertos%20y%20adaptadores.md))

**Adaptador:** implementación que conecta una tecnología concreta con un puerto. ([Puertos y adaptadores](02-Puertos%20y%20adaptadores.md))

**Arquitectura en capas (N-capas):** organización en presentación, negocio y datos, donde cada capa depende de la de abajo. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**Capa pasamanos (pass-through):** capa que solo reenvía llamadas a la siguiente sin agregar comportamiento. ([Pruebas de arquitectura](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md))

**Caso de uso:** operación de la aplicación desde el punto de vista del usuario, que orquesta el dominio. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**Dominio:** modelo del negocio: entidades, value objects, reglas e invariantes. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**DTO de aplicación:** tipo que devuelve un caso de uso; los adaptadores de entrada no ven entidades del dominio. ([Estructura en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md))

**Erosión de la arquitectura:** degradación gradual de las reglas de diseño por cambios pequeños que nadie revisa en conjunto. ([Pruebas de arquitectura](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md))

**Infraestructura:** detalles técnicos como base de datos, archivos, HTTP, colas y email. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**internal:** modificador que limita un tipo a su propio proyecto; oculta los detalles de infraestructura. ([Estructura en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md))

**Inversión de dependencias (DIP):** principio por el que los módulos de alto nivel no dependen de los de bajo nivel; ambos dependen de abstracciones que pertenecen al de alto nivel. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**Método de extensión de registro:** método `AddAplicacion()`/`AddInfraestructura()` con el que cada capa registra sus servicios en DI. ([Estructura en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md))

**Monolito modular:** aplicación desplegable única dividida en módulos del negocio con fronteras y dependencias controladas. ([Pruebas de arquitectura](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md))

**NetArchTest:** biblioteca para escribir pruebas de arquitectura en .NET con una API fluida. ([Pruebas de arquitectura](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md))

**Prueba de arquitectura:** prueba automática que verifica reglas estructurales del código, como dependencias o visibilidad. ([Pruebas de arquitectura](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md))

**Puerto:** interfaz definida por la aplicación para comunicarse con el exterior. ([Puertos y adaptadores](02-Puertos%20y%20adaptadores.md))

**Puerto de entrada (driving / primario):** forma en que el exterior le pide cosas a la aplicación; normalmente, los casos de uso. ([Puertos y adaptadores](02-Puertos%20y%20adaptadores.md))

**Puerto de salida (driven / secundario):** lo que la aplicación necesita del exterior, como persistencia, pagos o mensajería. ([Puertos y adaptadores](02-Puertos%20y%20adaptadores.md))

**Raíz de composición (composition root):** único lugar donde se conectan las implementaciones concretas con sus interfaces, en la capa más externa. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**Referencia de proyecto:** dependencia declarada en el `.csproj`; el compilador impide usar tipos de proyectos no referenciados. ([Estructura en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md))

**Regla de dependencias:** el código fuente solo puede depender de capas más internas, nunca de las externas. ([Regla de dependencias](01-Capas%20y%20la%20regla%20de%20dependencias.md))

**Vertical Slice Architecture:** organización por funcionalidad completa (endpoint, lógica y datos juntos) en lugar de por capas técnicas. ([Pruebas de arquitectura](04-Pruebas%20de%20arquitectura%20y%20cuando%20no%20usar%20Clean.md))
