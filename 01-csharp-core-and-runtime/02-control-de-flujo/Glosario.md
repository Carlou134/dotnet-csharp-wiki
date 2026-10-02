# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Array (arreglo).** Colección de tamaño fijo de elementos del mismo tipo, accesibles por un índice que empieza en 0.

**Array escalonado (*jagged*).** Array de arrays, donde cada fila puede tener un largo distinto: `int[][]`.

**Array multidimensional.** Array rectangular con varias dimensiones, como una tabla: `int[,]`.

**Bloque.** Grupo de sentencias entre llaves `{ }`.

**Booleano (`bool`).** Tipo con solo dos valores: `true` y `false`.

**`break`.** Sentencia que termina el bucle (o el caso de `switch`) actual.

**Bucle (*loop*).** Estructura que repite un bloque de código.

**Bucle infinito.** Bucle cuya condición nunca se vuelve `false`. Puede ser intencional (`while (true)` con `break`) o un error.

**Colección.** Cualquier grupo de elementos que se puede recorrer: arrays, listas, strings, diccionarios.

**Condición de parada.** Condición que, al volverse `false`, termina un bucle.

**`continue`.** Sentencia que salta el resto del cuerpo del bucle y pasa a la siguiente iteración.

**Cortocircuito.** Comportamiento de `&&` y `||`: si el primer operando ya determina el resultado, el segundo no se evalúa.

**`default` (en `switch`).** Caso que se ejecuta cuando ningún `case` coincide. Es opcional.

**Descarte (`_`).** En una expresión `switch`, el patrón que coincide con cualquier valor.

**Elemento.** Cada valor guardado dentro de un array o una colección.

**Error por uno (*off-by-one*).** Error típico de los bucles: recorrer una vez de más o de menos por usar `<=` en lugar de `<`, o al revés.

**Expresión booleana.** Expresión que produce un `bool`, como `edad >= 18`.

**Expresión `switch`.** Forma de `switch` que devuelve un valor: `x switch { 1 => "uno", _ => "otro" }`. Desde C# 8.

**Flujo de control.** Orden en que se ejecutan las instrucciones de un programa.

**Guarda (*guard clause*).** Comprobación al inicio de un bloque que sale antes (`return`) si algo no es válido, para evitar el anidamiento.

**Índice.** Posición de un elemento en un array. Va de `0` a `Length - 1`.

**Iteración.** Cada repetición de un bucle.

**Leyes de De Morgan.** Reglas para negar condiciones compuestas: `!(A && B)` es `!A || !B`, y `!(A || B)` es `!A && !B`.

**Operador de comparación.** Operador que compara dos valores y devuelve `bool`: `==`, `!=`, `<`, `>`, `<=`, `>=`.

**Operador lógico.** Operador que combina booleanos: `&&` (Y), `||` (O), `!` (NO), `^` (O exclusivo).

**Operador ternario.** `condición ? valorSiTrue : valorSiFalse`. Es una expresión: produce un valor.

**Patrón.** Forma de describir qué valores coinciden en un `is` o un `switch`: una constante, un rango (`>= 18`), un tipo.

**Predicado.** Función que recibe un elemento y devuelve `bool`. Se usa en `Array.Find` o `Array.Exists`.

**Rango (`..`).** Sintaxis para obtener una porción de un array o string: `a[1..3]`.

**`return`.** Sentencia que termina el método actual, aunque esté dentro de bucles.

**Sentencia condicional.** Estructura que ejecuta un bloque solo si se cumple una condición: `if`, `switch`.

**`switch`.** Estructura que compara un valor contra varios casos (`case`).

**Tabla de verdad.** Tabla que muestra el resultado de un operador lógico para cada combinación de entradas.

**Variable de control (iterador).** Variable que lleva la cuenta de las vueltas de un bucle, como `i` en un `for`.

**XOR (`^`).** O exclusivo: devuelve `true` si los dos operandos son distintos.
