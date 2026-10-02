# Asincronía y archivos

En esta carpeta aprendes a trabajar con **operaciones que tardan** sin bloquear tu programa: `async`/`await` y `Task`, cómo ejecutar varias operaciones a la vez, **cancelarlas** y manejar sus **errores**; y a **leer y escribir archivos**, desde los bytes de un `FileStream` hasta los métodos de una línea de `File`, con rutas portables y la codificación correcta.

-----

## Antes de empezar

Conviene que ya tengas:

* Genéricos (`Task<T>`): [Genéricos](../05-tipos-avanzados/04-Genericos.md).
* Iteradores y ejecución diferida: [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md).
* Excepciones, `using` e `IDisposable`: [Excepciones](../08-excepciones/README.md).
* Delegados y lambdas: [Delegados](../09-delegados-y-eventos/01-Delegados.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Programación asíncrona](01-Programacion%20asincrona.md) | `async`, `await`, `Task`, iniciar y esperar, E/S frente a CPU, `.Result` y `async void` | Genéricos, excepciones |
| [2. Concurrencia, cancelación y errores](02-Concurrencia%20cancelacion%20y%20errores.md) | `WhenAll`, `WhenAny`, tiempos límite, `CancellationToken`, excepciones en tareas y `IAsyncEnumerable` | Programación asíncrona |
| [3. Archivos y streams](03-Archivos%20y%20streams.md) | `Stream`, `FileStream`, `FileMode`/`FileAccess`, leer y escribir bytes, `Flush`, `Seek` | Excepciones, `using` |
| [4. Archivos de texto y la clase File](04-Archivos%20de%20texto%20y%20la%20clase%20File.md) | `File`, `ReadLines`, `Path`, `Directory`, `FileInfo`, `StreamReader`/`StreamWriter` y codificaciones | Archivos y streams |

-----

## El mapa completo en una mirada

```
async Task<T> XAsync()     método asíncrono que devolverá un T
await tarea                espera SIN bloquear el hilo y obtiene el resultado
Iniciar y esperar después  var t = XAsync(); ...; await t;   → varias a la vez
Nunca                      .Result · .Wait() · Thread.Sleep · async void (salvo eventos)

Task.WhenAll(a, b, c)      espera todas (resultados en orden)
Task.WhenAny(a, b)         la primera que termina (las demás siguen)
CancellationToken          cts.Token → pasarlo hacia abajo → ThrowIfCancellationRequested
Excepciones                aparecen al hacer await: try alrededor del await

Stream / FileStream        bytes: Read (devuelve cuántos) · Write · Flush · Seek · SetLength
FileMode                   Create · CreateNew · Open · OpenOrCreate · Append · Truncate
File                       ReadAllText · WriteAllText · AppendAllText · ReadLines (perezoso)
Path / Directory           Path.Combine · Directory.CreateDirectory · EnumerateFiles
StreamReader/Writer        texto línea a línea, UTF-8 por defecto
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con código chico y explicado.
5. **Ejemplo completo:** todo junto en un programa que compila.
6. **Errores comunes:** lo que suele salir mal, por qué pasa, cómo se arregla y el código de error o la excepción.
7. **Según la versión de C#:** qué cambió entre versiones, para reconocer código antiguo y moderno.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento frecuentes.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya conoces las herramientas centrales del lenguaje. El último módulo trata de cómo escribir código que otros (y tú mismo en seis meses) puedan entender y cambiar: [Código limpio](../11-codigo-limpio/README.md).
