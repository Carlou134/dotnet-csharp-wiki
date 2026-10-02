# Domain-Driven Design (DDD) táctico

En esta carpeta aprendes a escribir un **modelo de dominio** que protege las reglas del negocio en lugar de dejarlas dispersas en servicios: la diferencia entre un modelo **anémico** y uno **rico**, el **lenguaje ubicuo** y los tres bloques tácticos básicos de DDD: **entidades**, **value objects** y **agregados**.

-----

## Antes de empezar

Conviene que ya tengas:

* Encapsulación, constructores y herencia: [POO](../../01-csharp-core-and-runtime/04-poo/README.md).
* Records e igualdad por valor: [Structs y records](../../01-csharp-core-and-runtime/05-tipos-avanzados/02-Structs%20y%20records.md).
* Excepciones: [Excepciones](../../01-csharp-core-and-runtime/08-excepciones/README.md).
* SOLID e inyección de dependencias: [Código limpio](../../01-csharp-core-and-runtime/11-codigo-limpio/README.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Modelo anémico y modelo de dominio](01-Modelo%20anemico%20y%20modelo%20de%20dominio.md) | Qué es DDD, modelo anémico frente a rico, invariantes, lenguaje ubicuo, bloques tácticos | POO, excepciones |
| [2. Entidades](02-Entidades.md) | Identidad, igualdad por Id (`Equals`, `GetHashCode`, `==`), clase base `Entidad`, factories | Modelo de dominio |
| [3. Value objects](03-Value%20objects.md) | Inmutabilidad, igualdad por valor con `record`, auto-validación, obsesión por los primitivos, `Dinero` y `Email` | Entidades, records |
| [4. Agregados](04-Agregados.md) | Raíz del agregado, fronteras, `AsReadOnly()`, referencias por Id, repositorio por agregado, tamaño | Entidades, value objects |

Para practicar más: [Ejercicios de DDD](Ejercicios.md), con los 8 ejercicios de la Sesión 8 corregidos y 3 retos (todos con la solución plegada).

-----

## El mapa completo en una mirada

```
Modelo anémico      solo { get; set; } + lógica en servicios       → reglas dispersas, estados inválidos
Modelo rico         private set + métodos del negocio que validan  → el objeto protege sus invariantes
Lenguaje ubicuo     pedido.Confirmar()  en vez de  pedido.Estado = 2

Entidad             identidad (Id inmutable) · cambia en el tiempo · igualdad por Id · class
Value object        sin identidad · inmutable · igualdad por valor · se auto-valida · record
Agregado            grupo que cambia junto · se accede SOLO por la raíz · una transacción
                    colecciones con AsReadOnly() · otros agregados por Id · un repositorio por agregado

¿Entidad o VO?      ¿me importa CUÁL es (entidad) o solo CÓMO es (value object)?
¿Mismo agregado?    ¿hay una regla que exige que cambien en la MISMA transacción?
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura que en [C# Core](../../01-csharp-core-and-runtime/README.md): en una frase, antes de empezar, el problema, cómo funciona, ejemplo completo, errores comunes, según la versión de C#, cuándo sí y cuándo no, resumen en 5 líneas, para profundizar, en entrevista, práctica y siguiente lección.

-----

## Después de esta carpeta

Con el dominio modelado, el siguiente paso es organizar el resto de la aplicación a su alrededor: capas, puertos y adaptadores (Clean y Hexagonal Architecture), repositorios con EF Core y eventos de dominio. Para comunicar agregados y servicios con eventos: [Arquitectura dirigida por eventos](../02-event-driven-architecture/README.md). Para exponer el dominio por HTTP: [Diseño de APIs REST](../../03-aspnet-core-apis/01-diseno-de-apis-rest/README.md).
