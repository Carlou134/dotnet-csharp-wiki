# Métodos

En esta carpeta aprendes a **dividir un programa en piezas reutilizables con nombre**: a definir y llamar métodos con parámetros, a hacerlos flexibles con **parámetros opcionales, argumentos con nombre y sobrecarga**, a **devolver resultados** (con `return`, `out` y tuplas) y a escribir **expresiones lambda** para pasar comportamiento como si fuera un dato. Es el puente entre los programas de un solo bloque y la programación orientada a objetos.

-----

## Antes de empezar

Conviene que ya tengas:

* Variables, condicionales, arrays y bucles: [Control de flujo](../02-control-de-flujo/README.md).
* La diferencia entre tipos de valor y de referencia, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md), porque explica qué pasa con los argumentos.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Definir y llamar métodos](01-Definir%20y%20llamar%20metodos.md) | Definición, llamada, parámetros frente a argumentos, `void`, ámbito y `static` | Control de flujo |
| [2. Parámetros opcionales, nombrados y sobrecarga](02-Parametros%20opcionales%20nombrados%20y%20sobrecarga.md) | Valores por defecto, argumentos con nombre, sobrecarga, firma y `params` | Definir y llamar métodos |
| [3. Valores de retorno y parámetros out](03-Valores%20de%20retorno%20y%20parametros%20out.md) | `return`, tipo de retorno, `out`, `ref`, `in`, tuplas y deconstrucción | Parámetros opcionales y sobrecarga |
| [4. Expresiones lambda](04-Expresiones%20lambda.md) | Cuerpo de expresión, métodos como argumentos, lambdas, `Func`, `Action`, `Predicate` y captura | Valores de retorno |

-----

## El mapa completo en una mirada

```
Definir         ->  static int Sumar(int a, int b) { return a + b; }
Llamar          ->  int r = Sumar(2, 3);           (sin paréntesis no hay llamada)
Parámetro       ->  la variable en la definición   (a, b)
Argumento       ->  el valor en la llamada         (2, 3)

Flexibilidad:
Opcional        ->  void M(string s, int n = 1)    "puedo omitir n"
Con nombre      ->  M(n: 5, s: "x")                "el orden no importa"
Sobrecarga      ->  Sumar(int,int) · Sumar(double,double)   "mismo nombre, otros parámetros"
params          ->  SumarTodos(params int[] xs)    "cantidad variable"

Salida:
return          ->  devuelve UN valor y termina el método
out             ->  bool TryX(..., out T resultado)  "éxito + resultado"
Tupla           ->  (int Min, int Max) M() → var (min, max) = M();

Lambdas:
Cuerpo expr.    ->  static int Doble(int x) => x * 2;
Lambda          ->  n => n % 2 == 0                 "método anónimo en línea"
Delegados       ->  Func<in..., out> · Action<in...> · Predicate<T>
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

Ya sabes organizar la lógica en métodos. El siguiente paso es agrupar esos métodos **junto con los datos que manipulan** en tipos propios: [Programación orientada a objetos](../04-poo/README.md).
