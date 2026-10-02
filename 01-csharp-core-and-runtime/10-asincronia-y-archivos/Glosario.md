# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**`AggregateException`.** Excepción que agrupa varias excepciones; es el tipo de `Task.Exception`.

**Asíncrono.** Se inicia una operación y, mientras se completa, el hilo queda libre para otra cosa.

**`async`.** Modificador que habilita `await` dentro de un método; el método devuelve `Task`, `Task<T>` o (solo en eventos) `void`.

**`await`.** Espera a que una tarea termine sin bloquear el hilo y obtiene su resultado (o lanza su excepción).

**Búfer (*buffer*).** Zona de memoria intermedia donde se acumulan datos antes de leerlos o escribirlos.

**`CancellationToken`.** Objeto que recibe la señal de cancelación; se pasa a los métodos cancelables.

**`CancellationTokenSource`.** Objeto que emite la señal de cancelación (`Cancel`, `CancelAfter`).

**Codificación (*encoding*).** Regla que convierte caracteres en bytes y viceversa: UTF-8, ASCII, Latin-1.

**Concurrencia.** Varias operaciones en curso al mismo tiempo, aunque no necesariamente ejecutándose en el mismo instante.

**Deadlock (bloqueo mutuo).** Situación en la que dos partes se esperan entre sí y ninguna avanza; típica al usar `.Result` o `.Wait()` en código asíncrono.

**Directorio de trabajo.** Carpeta desde la que se resuelven las rutas relativas (`Environment.CurrentDirectory`).

**E/S (*I/O-bound*).** Operación que espera por algo externo: red, disco, base de datos.

**`File` / `FileInfo`.** Clase estática con operaciones de archivo de una línea / objeto que representa un archivo concreto.

**`FileAccess`.** Para qué se abre un archivo: `Read`, `Write` o `ReadWrite`.

**`FileMode`.** Qué hacer al abrir un archivo según exista o no: `Create`, `CreateNew`, `Open`, `OpenOrCreate`, `Append`, `Truncate`.

**`FileStream`.** Stream que lee y escribe bytes de un archivo.

**`Flush`.** Forzar la escritura en el destino de lo que quedó en el búfer.

**Hilo (*thread*).** Secuencia de ejecución; un programa puede tener varios.

**`OperationCanceledException`.** Excepción que señala que una operación fue cancelada.

**Paralelismo.** Varias operaciones ejecutándose literalmente a la vez en distintos núcleos.

**`Path`.** Clase estática para construir y analizar rutas de forma portable.

**Posición (`Position`) / `Seek`.** Punto del stream donde ocurrirá la próxima lectura o escritura / método para moverlo.

**Sincrónico (bloqueante).** Cada operación termina antes de que empiece la siguiente; mientras espera, el hilo no hace nada más.

**Stream (flujo).** Secuencia de bytes con operaciones para leer, escribir y, a veces, moverse.

**`StreamReader` / `StreamWriter`.** Adaptadores que leen y escriben texto sobre un stream, con una codificación.

**TAP (*Task-based Asynchronous Pattern*).** El patrón asíncrono de .NET basado en `Task`, `async` y `await`.

**`Task` / `Task<T>`.** Objeto que representa una operación en curso, sin resultado o con un resultado de tipo `T`.

**`Task.WhenAll` / `Task.WhenAny`.** Tareas que terminan cuando terminan todas las indicadas / la primera de ellas.

**Thread pool.** Conjunto de hilos que .NET reutiliza para ejecutar tareas; `Task.Run` envía trabajo ahí.
