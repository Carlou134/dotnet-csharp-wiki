# Delegados y eventos

En esta carpeta aprendes a **tratar el comportamiento como un dato**: los **delegados** guardan referencias a métodos para pasarlos, cambiarlos y combinarlos (multicast), y los **eventos** construyen sobre ellos el patrón publicador-suscriptor, para que un objeto avise a otros que algo ocurrió sin conocerlos.

-----

## Antes de empezar

Conviene que ya tengas:

* Lambdas, `Func`, `Action` y `Predicate`: [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).
* POO completa (estáticos, herencia, `protected virtual`, interfaces): [POO](../04-poo/README.md).
* `IDisposable` y `using`, mencionados en [Manejo de excepciones](../08-excepciones/01-Manejo%20de%20excepciones.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Delegados](01-Delegados.md) | Tipos delegados, grupos de métodos, `Func`/`Action`/`Predicate`, multicast y estrategias | Expresiones lambda, POO |
| [2. Eventos](02-Eventos.md) | `event`, publicador y suscriptor, `EventHandler<T>`, `EventArgs`, `OnX`, eventos en interfaces y fugas de memoria | Delegados |

-----

## El mapa completo en una mirada

```
Delegado        ->  delegate string Proceso(string s);       "tipo para métodos con esta firma"
Asignar         ->  Proceso p = Metodo;     (sin paréntesis)  ·  p = s => s.ToUpper();
Invocar         ->  p("hola")  ·  p.Invoke("hola")  ·  p?.Invoke("hola")
Genéricos       ->  Func<in..., out TResult>  ·  Action<in...>  ·  Predicate<T>
Multicast       ->  += agrega · -= quita · se ejecutan en orden · retorno = el del último

Evento          ->  public event EventHandler<TArgs>? Algo;
Publicador      ->  protected virtual void OnAlgo(TArgs e) => Algo?.Invoke(this, e);
Suscriptor      ->  publicador.Algo += Manejador;   ...   publicador.Algo -= Manejador;
Desde afuera    ->  solo += y -=  (no se puede asignar ni disparar)
Sincrónico      ->  los manejadores se ejecutan uno tras otro, en el mismo hilo
Fugas           ->  el publicador referencia a los suscriptores: desuscribirse en Dispose
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

Ya sabes reaccionar a lo que ocurre de forma sincrónica. El siguiente paso es trabajar con operaciones que tardan (red, disco) sin bloquear el programa: [Asincronía y archivos](../10-asincronia-y-archivos/README.md).
