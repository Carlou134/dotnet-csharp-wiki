# Tipos y variables

En esta carpeta aprendes **cómo C# representa los datos**: a declarar variables con su tipo, a distinguir **tipos de valor y de referencia** (y qué pasa en memoria con cada uno), a **convertir** entre tipos sin perder datos por accidente, a operar con **números** y a trabajar con **texto**. Todo programa mueve datos; esta es la base para todo lo que sigue.

-----

## Antes de empezar

Conviene que ya tengas:

* Un entorno funcionando y la capacidad de ejecutar un programa de consola: [Introducción](../00-introduccion/README.md).
* `Console.WriteLine` y `Console.ReadLine`, de [Tu primer programa](../00-introduccion/03-Tu%20primer%20programa.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Variables y tipos de datos](01-Variables%20y%20tipos%20de%20datos.md) | Declarar, inicializar, tipos integrados, `var`, `const`, valores por defecto y convenciones de nombres | Tu primer programa |
| [2. Tipos de valor y de referencia](02-Tipos%20de%20valor%20y%20de%20referencia.md) | Copia por valor frente a copia de referencia, `null`, stack, heap y GC | Variables y tipos |
| [3. Conversiones de tipos](03-Conversiones%20de%20tipos.md) | Conversión implícita, cast, `Parse`, `TryParse`, `Convert` y cultura | Variables y tipos |
| [4. Números y operadores](04-Numeros%20y%20operadores.md) | `int`/`double`/`decimal`, división entera, `%`, `++`, precedencia, `Math` y `Random` | Conversiones |
| [5. Texto: char y string](05-Texto%20char%20y%20string.md) | Literales, escapes, verbatim, raw strings, concatenación e interpolación | Tipos de valor y de referencia |
| [6. Métodos de string y StringBuilder](06-Metodos%20de%20string%20y%20StringBuilder.md) | Buscar, extraer, transformar, dividir, comparar y construir texto eficientemente | Texto: char y string |

-----

## El mapa completo en una mirada

```
Declarar        ->  int edad = 30;   var nombre = "Ana";   const int Max = 10;
Tipos clave     ->  int (enteros)  double (decimales)  decimal (dinero)  bool  char  string

Valor           ->  la variable ES el dato        (int, double, bool, struct)  → se copia el dato
Referencia      ->  la variable APUNTA al dato    (class, string, arrays)      → se copia la referencia
Memoria         ->  locales en el stack, objetos con new en el heap, el GC limpia el heap

Convertir       ->  implícita (sin pérdida)  |  (int)x trunca  |  TryParse para texto
Números         ->  5 / 2 = 2  ·  5 / 2.0 = 2.5  ·  % = resto  ·  Math.Pow, Math.Round
Texto           ->  "normal\n"  @"C:\ruta"  """raw"""  $"Hola {nombre}"
Strings         ->  inmutables: los métodos devuelven uno nuevo  ·  StringBuilder en bucles
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

Ya sabes representar y transformar datos. El siguiente paso es tomar decisiones con ellos y repetir acciones: [Control de flujo](../02-control-de-flujo/README.md).
