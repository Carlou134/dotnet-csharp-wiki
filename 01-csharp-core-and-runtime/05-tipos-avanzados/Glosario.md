# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Argumento de tipo.** El tipo concreto que se usa en lugar de un parámetro de tipo: `int` en `List<int>`.

**Contravarianza (`in`).** Propiedad de un parámetro de tipo que solo se recibe: permite usar `I<Base>` donde se espera `I<Derivado>`. Ejemplo: `IComparer<in T>`.

**Covarianza (`out`).** Propiedad de un parámetro de tipo que solo se devuelve: permite usar `I<Derivado>` donde se espera `I<Base>`. Ejemplo: `IEnumerable<out T>`.

**`default`.** Valor por defecto de un tipo: `0`, `false` o `null`. En genéricos, `default` funciona sin saber si `T` es de valor o de referencia.

**DTO (*Data Transfer Object*).** Objeto cuyo único propósito es transportar datos entre capas o sistemas. Suele modelarse con un `record`.

**Enumeración (`enum`).** Tipo de valor que define un conjunto de constantes con nombre, respaldadas por un entero.

**Expresión `with`.** Crea una copia de un record (o struct) cambiando algunas propiedades: `p with { Edad = 31 }`.

**`[Flags]`.** Atributo que indica que los miembros de un enum son bits combinables con `|`, `&` y `~`.

**Genérico.** Clase, interfaz, método o delegado con uno o más parámetros de tipo.

**Igualdad por valor.** Dos instancias son iguales si sus datos son iguales. Es la igualdad de los records.

**Inferencia de tipos.** El compilador deduce el argumento de tipo de un método genérico a partir de sus argumentos.

**Invarianza.** Solo se acepta exactamente el mismo argumento de tipo. Las clases genéricas como `List<T>` son invariantes.

**`Nullable<T>`.** Struct que permite `null` en un tipo de valor. `int?` es su forma corta.

**Operador condicional de null (`?.`, `?[]`).** Accede a un miembro o índice solo si el objeto no es `null`; si lo es, el resultado es `null`.

**Operador de fusión de null (`??`).** Devuelve el operando derecho si el izquierdo es `null`. `??=` asigna solo si la variable es `null`.

**Operador que perdona null (`!`).** Suprime una advertencia de posible `null`. No comprueba nada en ejecución.

**Operadores elevados (*lifted*).** Operadores que funcionan con tipos anulables y propagan el `null`: `5 + null` es `null`.

**Parámetro de tipo.** Marcador (`T`, `TKey`) que representa un tipo que se define al usar el genérico.

**`readonly struct`.** Struct cuyos campos no pueden cambiar después de construirse.

**Record.** Tipo orientado a datos con igualdad por valor, `ToString`, `Deconstruct` y `with` generados. `record` es de referencia; `record struct`, de valor.

**Record posicional.** Record declarado con parámetros: `record Punto(int X, int Y)`. Sus propiedades se generan automáticamente.

**Restricción genérica (*constraint*).** Condición que debe cumplir un argumento de tipo, declarada con `where`: `where T : class, new()`.

**`struct`.** Tipo de valor definido por el usuario: se copia al asignarlo, no admite herencia y no es `null`.

**Tipo de referencia que acepta null (NRT).** Anotación `string?` que indica que una referencia puede ser `null`, para que el compilador avise. No cambia nada en ejecución.

**Tipo genérico abierto / cerrado.** `List<T>` o `List<>` (sin tipo concreto) frente a `List<int>` (con tipo concreto).

**Tipo subyacente.** El tipo entero que guarda el valor de un enum (`int` por defecto).
