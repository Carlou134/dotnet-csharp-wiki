# Control de flujo

En esta carpeta aprendes a **tomar decisiones y repetir acciones**: a escribir condiciones con **lógica booleana**, a elegir caminos con **condicionales** (`if`, `switch`, ternario), a agrupar datos en **arrays** y a recorrerlos con **bucles** (`while`, `for`, `foreach`). Con esto, tus programas dejan de ser una lista fija de instrucciones y reaccionan a los datos.

-----

## Antes de empezar

Conviene que ya tengas:

* Variables, tipos numéricos y strings: [Tipos y variables](../01-tipos-y-variables/README.md).
* Conversiones con `TryParse`, de [Conversiones de tipos](../01-tipos-y-variables/03-Conversiones%20de%20tipos.md), porque los ejemplos leen datos del usuario.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Lógica booleana](01-Logica%20booleana.md) | `bool`, comparaciones, `&&`, `\|\|`, `!`, `^`, cortocircuito y patrones `is` | Números y operadores |
| [2. Condicionales](02-Condicionales.md) | `if`/`else if`/`else`, guardas, `switch`, ternario, expresión `switch` y `??` | Lógica booleana |
| [3. Arrays](03-Arrays.md) | Crear, indexar, modificar y ordenar arrays; rangos, `Array.Find` y multidimensionales | Condicionales |
| [4. Bucles](04-Bucles.md) | `while`, `do...while`, `for`, `foreach`, `break`, `continue`, `return` y anidados | Arrays |

Arrays va antes que bucles porque `foreach` necesita algo que recorrer, y los ejemplos de arrays solo usan un `for` sencillo.

-----

## El mapa completo en una mirada

```
Comparar        ->  ==  !=  <  >  <=  >=           → producen bool
Combinar        ->  &&  ||  !  ^                    → && y || hacen cortocircuito
Patrones        ->  x is >= 0 and <= 10   ·   x is not null

Decidir         ->  if / else if / else             "tomo un camino"
                    switch (x) { case ...: break; }  "un valor contra muchos casos"
                    c ? a : b   ·   x switch { ... => ... }   "elijo un VALOR"

Agrupar         ->  int[] a = { 1, 2, 3 };   a[0]   a[^1]   a.Length   (tamaño fijo)

Repetir         ->  while (c)        "mientras..."
                    do { } while (c); "al menos una vez"
                    for (i...)       "n veces / necesito el índice"
                    foreach (x in a) "por cada elemento"
Saltar          ->  break (sale del bucle) · continue (siguiente vuelta) · return (sale del método)
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

Tus programas ya deciden y repiten, pero todo vive en un solo bloque de código. El siguiente paso es dividirlo en piezas reutilizables con nombre: [Métodos](../03-metodos/README.md).
