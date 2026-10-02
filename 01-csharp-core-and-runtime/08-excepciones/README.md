# Excepciones

En esta carpeta aprendes a **manejar los errores que ocurren en ejecución**: a capturarlos con `try`/`catch`/`finally`, a entender cómo **se propagan** por la pila de llamadas, a filtrarlos con `when`, y a **lanzar** tus propios errores con las excepciones adecuadas o con **excepciones personalizadas**, sabiendo cuándo conviene una excepción y cuándo un resultado.

-----

## Antes de empezar

Conviene que ya tengas:

* Métodos y la pila de llamadas: [Métodos](../03-metodos/README.md).
* Herencia, porque todas las excepciones heredan de `Exception`: [Herencia](../04-poo/06-Herencia.md).
* El patrón `TryX` y los tipos que aceptan null: [Tipos que aceptan null](../05-tipos-avanzados/03-Tipos%20que%20aceptan%20null.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Manejo de excepciones](01-Manejo%20de%20excepciones.md) | `try`, `catch`, `finally`, `using`, excepciones habituales, propagación y filtros `when` | Métodos, herencia |
| [2. Lanzar y crear excepciones](02-Lanzar%20y%20crear%20excepciones.md) | `throw`, guardas, `ThrowIf...`, `throw;` frente a `throw ex;`, `InnerException`, excepciones propias y patrón Result | Manejo de excepciones |

-----

## El mapa completo en una mirada

```
try { ... }                     código que puede fallar
catch (TipoEspecifico ex)       maneja ESE tipo (de lo específico a lo general)
catch (Tipo ex) when (cond)     solo si se cumple la condición (no pierde la traza)
finally { ... }                 se ejecuta SIEMPRE (limpieza)
using var x = ...;              finally automático para IDisposable

Propagación     ->  sin catch, la excepción sube al método que llamó... hasta terminar el programa

throw new ArgumentNullException(nameof(x));     guardas al inicio (fail fast)
ArgumentNullException.ThrowIfNull(x);           atajo moderno
x ?? throw new ...                              throw como expresión
throw;                                          relanzar conservando la traza (NO throw ex;)
throw new MiException("contexto", ex);          envolver con InnerException
class MiErrorException : Exception { }          excepción propia

Excepción → lo anormal          TryX / Result → lo esperado
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con código chico y explicado.
5. **Ejemplo completo:** todo junto en un programa que compila.
6. **Errores comunes:** lo que suele salir mal, por qué pasa, cómo se arregla y el código de error del compilador.
7. **Según la versión de C#:** qué cambió entre versiones, para reconocer código antiguo y moderno.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento frecuentes.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes manejar errores. El siguiente paso es tratar el comportamiento como un dato y reaccionar a lo que ocurre: [Delegados y eventos](../09-delegados-y-eventos/README.md).
