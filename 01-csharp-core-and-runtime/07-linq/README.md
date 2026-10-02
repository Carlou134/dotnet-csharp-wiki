# LINQ

En esta carpeta aprendes a **consultar y transformar colecciones de forma declarativa** con LINQ: las dos sintaxis (de método y de consulta) y la **ejecución diferida**, los operadores para **filtrar, ordenar, paginar y proyectar**, y los que **resumen, agrupan y combinan** secuencias. Es una de las herramientas que más vas a usar en C#, en memoria y con bases de datos.

-----

## Antes de empezar

Conviene que ya tengas:

* Lambdas, `Func` y `Predicate`: [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).
* Colecciones e iteradores: [Colecciones](../06-colecciones/README.md), en especial [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md), que explica la ejecución diferida.
* Records y tipos anulables: [Tipos avanzados](../05-tipos-avanzados/README.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Introducción a LINQ](01-Introduccion%20a%20LINQ.md) | `Where`, `Select`, sintaxis de método y de consulta, tipos anónimos, ejecución diferida y materialización | Colecciones, lambdas |
| [2. Filtrar, ordenar y paginar](02-Filtrar%20ordenar%20y%20paginar.md) | `OrderBy`/`ThenBy`, `Skip`/`Take`, `First`/`Single`, `Any`/`All`, `Distinct`, `SelectMany` y `Chunk` | Introducción a LINQ |
| [3. Agregar, agrupar y unir](03-Agregar%20agrupar%20y%20unir.md) | `Sum`, `Average`, `MaxBy`, `Aggregate`, `GroupBy`, `ToLookup`, `Join`, `GroupJoin`, `Zip` y conjuntos | Filtrar, ordenar y paginar |

-----

## El mapa completo en una mirada

```
Filtrar       ->  Where · Distinct / DistinctBy · OfType<T>
Transformar   ->  Select · SelectMany (aplanar)
Ordenar       ->  OrderBy / OrderByDescending → ThenBy / ThenByDescending
Paginar       ->  Skip · Take · TakeWhile · SkipWhile · TakeLast · Chunk
Un elemento   ->  First · FirstOrDefault · Single · Last · ElementAt
Preguntar     ->  Any · All · Contains
Resumir       ->  Count · Sum · Average · Min / Max · MinBy / MaxBy · Aggregate
Agrupar       ->  GroupBy (diferido) · ToLookup (inmediato)
Combinar      ->  Join (inner) · GroupJoin (left) · Zip · Concat · Union · Intersect · Except
Materializar  ->  ToList · ToArray · ToDictionary · ToHashSet

Diferidos     ->  los que devuelven secuencias: se ejecutan al recorrer, en cada recorrido
Inmediatos    ->  los que devuelven un valor (Count, First, Sum...) y los To...()
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con código chico y explicado.
5. **Ejemplo completo:** todo junto en un programa que compila.
6. **Errores comunes:** lo que suele salir mal, por qué pasa, cómo se arregla y el código de error o la excepción.
7. **Según la versión de C#:** qué cambió entre versiones, para reconocer código antiguo y moderno.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento frecuentes.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes consultar datos. Lo siguiente es manejar lo que sale mal mientras se procesan: [Excepciones](../08-excepciones/README.md).
