# Colecciones

En esta carpeta aprendes a **guardar y manejar grupos de datos** con las estructuras de .NET: **listas** que crecen, **diccionarios** que buscan por clave, **conjuntos** sin duplicados, **colas, pilas y colas de prioridad** que definen el orden de salida, cómo **elegir la colección adecuada** según el rendimiento y la API, y cómo crear tus propias secuencias con **iteradores y `yield`**.

-----

## Antes de empezar

Conviene que ya tengas:

* Arrays y bucles: [Control de flujo](../02-control-de-flujo/README.md).
* Lambdas y `Predicate<T>`: [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).
* Genéricos (`List<T>`, `T`): [Genéricos](../05-tipos-avanzados/04-Genericos.md).
* `Equals` y `GetHashCode`: [La clase Object](../04-poo/10-La%20clase%20Object.md), clave para entender diccionarios y conjuntos.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Listas](01-Listas.md) | `List<T>`: agregar, buscar, quitar, rangos, ordenar y cómo crece por dentro | Arrays, genéricos |
| [2. Diccionarios y conjuntos](02-Diccionarios%20y%20conjuntos.md) | `Dictionary`, `TryGetValue`, `HashSet`, operaciones de conjuntos y colecciones ordenadas | Listas, La clase Object |
| [3. Colas, pilas y listas enlazadas](03-Colas%20pilas%20y%20listas%20enlazadas.md) | `Queue`, `Stack`, `PriorityQueue` y `LinkedList` | Listas |
| [4. Elegir la colección adecuada](04-Elegir%20la%20coleccion%20adecuada.md) | Complejidad, interfaces (`IEnumerable`, `IReadOnlyList`), colecciones heredadas y especializadas | Las tres anteriores |
| [5. Iteradores y yield](05-Iteradores%20y%20yield.md) | `yield return`, ejecución diferida, secuencias infinitas, `IEnumerable`/`IEnumerator` | Elegir la colección |

-----

## El mapa completo en una mirada

```
List<T>            ->  ordenada, por índice, crece sola           Add · Insert · Remove · Find · Sort
Dictionary<K,V>    ->  clave → valor, búsqueda O(1)               dic[k] · TryGetValue · TryAdd
HashSet<T>         ->  únicos, pertenencia O(1)                   Add (bool) · UnionWith · IntersectWith
Sorted...          ->  lo mismo, pero ordenado (O(log n))
Queue<T>           ->  FIFO: primero en entrar, primero en salir   Enqueue · Dequeue · Peek
Stack<T>           ->  LIFO: último en entrar, primero en salir    Push · Pop · Peek
PriorityQueue      ->  sale primero la mejor prioridad
LinkedList<T>      ->  nodos enlazados: insertar junto a un nodo en O(1)

En tus APIs        ->  acepta IEnumerable<T> / IReadOnlyList<T>, expón solo lectura
Heredadas          ->  ArrayList, Hashtable: object, casts, boxing → no usar

yield return       ->  secuencia perezosa: se calcula al recorrer, de a un elemento
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

Ya sabes guardar datos y recorrerlos de forma perezosa. El siguiente paso es consultarlos y transformarlos de manera declarativa: [LINQ](../07-linq/README.md).
