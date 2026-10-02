# Código limpio

En esta carpeta aprendes a escribir código que **otros (y tú mismo en seis meses) puedan entender y cambiar con seguridad**: nombres y comentarios que comunican, cómo reconocer **code smells**, los principios **DRY, KISS y YAGNI**, el bajo acoplamiento, la **deuda técnica** y el **refactoring**, y un repaso de la **sintaxis moderna de C#** que elimina código ceremonial.

-----

## Antes de empezar

Conviene que ya tengas:

* Todo el recorrido anterior, en especial [POO](../04-poo/README.md) (interfaces y polimorfismo) y [Excepciones](../08-excepciones/README.md).
* Haber escrito algunos programas propios: estos temas se entienden mucho mejor cuando ya sufriste código difícil de cambiar.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Nombres, comentarios y code smells](01-Nombres%20comentarios%20y%20code%20smells.md) | Nombres que revelan la intención, comentarios útiles, números mágicos y catálogo de code smells | Métodos, POO, enums |
| [2. Principios DRY, KISS y refactoring](02-Principios%20DRY%20KISS%20y%20refactoring.md) | DRY, KISS, YAGNI, acoplamiento y cohesión, deuda técnica, refactoring y code review | Nombres y code smells, interfaces |
| [3. Sintaxis moderna de C#](03-Sintaxis%20moderna%20de%20CSharp.md) | Las novedades de C# 6 a C# 14 con ejemplos de antes y después | Todo el curso |

-----

## El mapa completo en una mirada

```
Nombres         ->  revelan la intención: verbos para métodos, sustantivos para clases, preguntas para bool
Comentarios     ->  el PORQUÉ (decisiones, restricciones), no el QUÉ; nunca código comentado
Números mágicos ->  constantes · enums · configuración
Code smells     ->  método largo · muchos parámetros · duplicación · anidamiento · switch repetido

DRY             ->  cada conocimiento en un solo lugar (pero regla de tres)
KISS / YAGNI    ->  la solución más simple para el problema de HOY
Acoplamiento    ->  depender de interfaces inyectadas, no de clases concretas
Deuda técnica   ->  atajos de hoy = intereses mañana
Refactoring     ->  mejorar la estructura SIN cambiar el comportamiento: pruebas + pasos pequeños + continuo

C# moderno      ->  $"..." · => · ?. · tuplas · switch · records · global using · constructores primarios · [..]
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con código chico y explicado.
5. **Ejemplo completo:** todo junto en un programa que compila.
6. **Errores comunes:** lo que suele salir mal, por qué pasa y cómo se arregla.
7. **Según la versión de C#:** qué cambió entre versiones, para reconocer código antiguo y moderno.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento frecuentes.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Terminaste el bloque de C# Core & Runtime. Los siguientes pasos naturales, según el plan del repositorio, son la arquitectura y los patrones de diseño (SOLID en profundidad, patrones GoF, Clean Architecture) y ASP.NET Core. Puedes empezar por modelar el dominio: [Domain-Driven Design táctico](../../02-architecture-and-design-patterns/01-domain-driven-design/README.md).
