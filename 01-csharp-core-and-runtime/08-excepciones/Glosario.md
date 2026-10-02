# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Capturar (*catch*).** Interceptar una excepción en un bloque `catch` para manejarla.

**Cláusula de guarda.** Validación al inicio de un método que lanza una excepción si los datos no sirven, para que el resto trabaje con datos válidos.

**Excepción.** Objeto que representa un error en tiempo de ejecución. Todas derivan de `System.Exception`.

**Excepción interna (*inner exception*).** Excepción original que otra envuelve para agregar contexto. Se consulta con `InnerException`.

**Excepción no controlada.** Excepción que nadie captura y llega al punto de entrada; termina el proceso.

**Excepción personalizada.** Clase propia que hereda de `Exception` para representar un error del dominio. Su nombre termina en `Exception`.

**Fail fast.** Principio de detectar y señalar un error lo antes posible, cerca de su origen.

**Filtro de excepción.** Condición `when` en un `catch`: solo maneja la excepción si se cumple.

**`finally`.** Bloque que se ejecuta siempre, haya o no excepción. Se usa para liberar recursos.

**Lanzar (*throw*).** Producir una excepción con `throw`.

**Patrón Result.** Devolver un objeto que indica éxito o error en lugar de lanzar una excepción, para fallos esperados.

**Pila de llamadas (*call stack*).** Cadena de métodos que se llamaron hasta el punto actual.

**Propagación.** Viaje de una excepción no capturada hacia el método que llamó, y así sucesivamente.

**Relanzar.** Volver a lanzar la excepción capturada con `throw;`, conservando su traza.

**Traza de pila (*stack trace*).** Registro de la cadena de llamadas en el momento en que ocurrió una excepción.

**`using`.** Sentencia que llama a `Dispose()` automáticamente al salir del bloque, aunque haya una excepción.
