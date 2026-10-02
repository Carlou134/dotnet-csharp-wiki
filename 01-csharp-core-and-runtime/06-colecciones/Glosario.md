# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**`Capacity`.** Espacio reservado internamente por una `List<T>`. Crece (normalmente al doble) cuando se llena.

**Clave (*key*).** Dato por el que se busca en un diccionario. Debe ser única, no `null` y no cambiar mientras está en la colección.

**Cola (*queue*).** Estructura FIFO: el primer elemento en entrar es el primero en salir. En .NET, `Queue<T>`.

**Cola de prioridad.** Cola que saca primero el elemento con mejor prioridad. En .NET, `PriorityQueue<TElement, TPriority>`.

**Colección concurrente.** Colección segura para que varios hilos la usen a la vez, como `ConcurrentDictionary`.

**Colección heredada (*legacy*).** Colección anterior a los genéricos que guarda `object`, como `ArrayList` o `Hashtable`.

**Colección inmutable.** Colección que no cambia; cada "modificación" devuelve una nueva.

**Complejidad (notación O).** Cómo crece el tiempo de una operación según la cantidad de elementos: O(1) constante, O(log n) muy lento crecimiento, O(n) proporcional, O(n²) cuadrático.

**Conjunto (*set*).** Colección sin duplicados. En .NET, `HashSet<T>` y `SortedSet<T>`.

**`Count`.** Cantidad de elementos de una colección (los arrays usan `Length`).

**Diccionario.** Colección de pares clave-valor con claves únicas. En .NET, `Dictionary<TKey, TValue>`.

**Ejecución diferida / evaluación perezosa.** El código de una secuencia no se ejecuta al definirla, sino al recorrerla, y se repite en cada recorrido.

**Expresión de colección.** Sintaxis `[1, 2, 3]` para crear arrays, listas, conjuntos y otras colecciones (C# 12).

**FIFO (*First In, First Out*).** "Primero en entrar, primero en salir". Comportamiento de una cola.

**`IEnumerable<T>`.** Interfaz de "algo que se puede recorrer". Tiene un único método: `GetEnumerator()`.

**`IEnumerator<T>`.** El cursor que recorre una secuencia: `MoveNext()`, `Current` y `Dispose()`.

**`IReadOnlyList<T>`.** Interfaz de solo lectura con `Count` e indexador. Útil para exponer colecciones sin permitir modificarlas.

**Iterador.** Método que usa `yield return` o `yield break` para producir una secuencia de forma perezosa.

**`KeyValuePair<TKey, TValue>`.** Un par clave-valor, como los que aparecen al recorrer un diccionario.

**LIFO (*Last In, First Out*).** "Último en entrar, primero en salir". Comportamiento de una pila.

**Lista enlazada.** Secuencia de nodos donde cada uno apunta al siguiente y al anterior. En .NET, `LinkedList<T>`.

**Máquina de estados.** Clase que genera el compilador para un iterador; recuerda en qué `yield` quedó y sus variables locales.

**Nodo.** Cada eslabón de una lista enlazada: un valor (`Value`) y enlaces a sus vecinos (`Next`, `Previous`).

**Operaciones de conjuntos.** Unión (todos), intersección (los comunes), diferencia (los de uno que no están en otro) y diferencia simétrica (los que están en uno solo).

**`Peek`.** Mirar el siguiente elemento de una cola o pila sin sacarlo.

**Pila (*stack*).** Estructura LIFO: el último elemento en entrar es el primero en salir. En .NET, `Stack<T>`.

**Rango.** Porción contigua de elementos: `GetRange(inicio, cantidad)` o `lista[1..3]`.

**Tabla hash.** Estructura que usa el `GetHashCode` de cada elemento para ubicarlo sin recorrer toda la colección. Base de `Dictionary` y `HashSet`.

**`yield break`.** Termina la secuencia de un iterador.

**`yield return`.** Entrega un elemento de un iterador y pausa el método hasta que se pida el siguiente.
