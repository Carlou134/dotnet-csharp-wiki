# Tipos avanzados

En esta carpeta aprendes a elegir y diseñar **el tipo correcto para cada dato**: **enums** para conjuntos fijos de opciones, **structs y records** para valores y datos con igualdad por valor, **tipos que aceptan null** para expresar (y controlar) la ausencia de valor, y **genéricos** con sus **restricciones y varianza** para escribir código reutilizable sin perder la seguridad de tipos.

-----

## Antes de empezar

Conviene que ya tengas:

* Programación orientada a objetos completa: [POO](../04-poo/README.md), en especial [La clase Object](../04-poo/10-La%20clase%20Object.md) (igualdad, `ToString`, boxing).
* La diferencia entre tipos de valor y de referencia: [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Enumeraciones](01-Enumeraciones.md) | `enum`, valores explícitos, `switch`, `Parse`/`TryParse`, `IsDefined` y `[Flags]` | Condicionales, POO |
| [2. Structs y records](02-Structs%20y%20records.md) | `struct`, `readonly struct`, `record`, `record struct`, `with` e igualdad por valor | La clase Object |
| [3. Tipos que aceptan null](03-Tipos%20que%20aceptan%20null.md) | `int?`, `??`, `?.`, `??=`, tipos de referencia anulables, `!` y null frente a "sin asignar" | Structs y records |
| [4. Genéricos](04-Genericos.md) | Clases, métodos e interfaces genéricas, inferencia, `default` y por qué evitan el boxing | Tipos que aceptan null |
| [5. Restricciones y varianza](05-Restricciones%20y%20varianza.md) | `where`, `new()`, `class`/`struct`, covarianza `out` y contravarianza `in` | Genéricos |

-----

## El mapa completo en una mirada

```
enum            ->  enum Estado { Pendiente, Pagado }      "opciones con nombre (un entero por dentro)"
[Flags]         ->  Leer | Escribir   ·   HasFlag   ·   &= ~X

struct          ->  tipo de VALOR propio: se copia al asignar
record          ->  datos con igualdad por valor, ToString y with   (referencia)
record struct   ->  lo mismo, pero tipo de valor

int?            ->  Nullable<int>: HasValue, Value, GetValueOrDefault
string?         ->  "puede ser null" (anotación para el compilador)
Operadores      ->  ??   ??=   ?.   ?[]   !

Genéricos       ->  class Caja<T>  ·  T Metodo<T>(T x)  ·  interface IRepo<T>
default         ->  0 / false / null según T
where           ->  where T : IComparable<T>, new()
Varianza        ->  out T (solo sale: IEnumerable)  ·  in T (solo entra: IComparer)
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

Los fragmentos de "Cómo funciona" pueden omitir partes; los de "Ejemplo completo" y "Práctica" compilan en un `Program.cs` (con los tipos declarados después de las sentencias).

-----

## Después de esta carpeta

Con genéricos ya puedes entender a fondo las estructuras de datos de .NET. El siguiente paso: [Colecciones](../06-colecciones/README.md).
