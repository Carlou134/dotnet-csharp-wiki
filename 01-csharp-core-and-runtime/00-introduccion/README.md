# Introducción a C# y .NET

En esta carpeta aprendes **qué son C# y .NET y cómo se relacionan**, a **preparar tu entorno** para compilar y ejecutar código, a escribir tu **primer programa de consola** (mostrar texto, leer al usuario, comentar) y a reconocer los **paradigmas de programación** que C# combina. Es el punto de partida de todo lo demás.

-----

## Antes de empezar

Conviene que ya tengas:

* Manejo básico de la computadora: crear carpetas, instalar programas y abrir una terminal.
* Ganas de escribir código y equivocarte: los errores del compilador son parte del aprendizaje.

No necesitas saber otro lenguaje de programación. Si ya conoces JavaScript o Python, encontrarás comparaciones útiles en algunas lecciones.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Qué es C# y .NET](01-Que%20es%20CSharp%20y%20.NET.md) | Lenguaje frente a plataforma, compilación a IL, CLR, JIT y versiones | Nada |
| [2. Preparar el entorno](02-Preparar%20el%20entorno.md) | Instalar el SDK, la CLI `dotnet`, el `.csproj`, editores y atajos | Qué es C# y .NET |
| [3. Tu primer programa](03-Tu%20primer%20programa.md) | `WriteLine`, `ReadLine`, secuencias de escape, top-level statements, comentarios | Preparar el entorno |
| [4. Paradigmas de programación](04-Paradigmas%20de%20programacion.md) | Estructurado, POO, funcional, eventos y AOP en C# | Tu primer programa |

-----

## El mapa completo en una mirada

```
Program.cs ──► compilador ──► .dll (IL) ──► CLR + JIT ──► código máquina

SDK            ->  "lo que necesito para desarrollar" (dotnet new / build / run)
Runtime        ->  "lo que necesito para ejecutar"
Console        ->  WriteLine / Write (salida), ReadLine (entrada, siempre string)
Comentarios    ->  //  /* */  ///

Paradigmas:
Estructurado   ->  secuencia + if/switch + bucles
POO            ->  clases y objetos (abstracción, encapsulación, herencia, polimorfismo)
Funcional      ->  funciones puras + LINQ + inmutabilidad
Eventos / AOP  ->  reaccionar a cambios / separar lo transversal
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
7. **Según la versión de C#:** qué cambió entre versiones, para reconocer código antiguo y moderno. Algunas lecciones la omiten cuando no aplica.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento frecuentes.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes compilar, ejecutar y escribir un programa que conversa con el usuario. El siguiente paso es entender cómo C# representa los datos: [Tipos y variables](../01-tipos-y-variables/README.md).
