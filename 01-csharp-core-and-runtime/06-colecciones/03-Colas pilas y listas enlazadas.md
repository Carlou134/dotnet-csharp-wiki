# Colas, pilas y listas enlazadas

## En una frase

`Queue<T>` (cola) saca los elementos en el **mismo orden en que entraron** (FIFO), `Stack<T>` (pila) saca **primero el último que entró** (LIFO), `PriorityQueue<T, P>` saca primero el de **mayor prioridad**, y `LinkedList<T>` es una cadena de nodos donde insertar o quitar en cualquier punto conocido es inmediato.

-----

## Antes de empezar

Conviene que ya sepas:

* Usar `List<T>` y por qué insertar al principio es costoso, de [Listas](01-Listas.md).
* El concepto de LIFO, que viste con el stack de memoria en [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Cola (*queue*):** estructura FIFO (*First In, First Out*): el primero en entrar es el primero en salir.
* **Pila (*stack*):** estructura LIFO (*Last In, First Out*): el último en entrar es el primero en salir.
* **Encolar / desencolar:** agregar al final de la cola / sacar del frente (`Enqueue` / `Dequeue`).
* **Apilar / desapilar:** poner en el tope / sacar del tope (`Push` / `Pop`).
* **Ver (`Peek`):** mirar el siguiente elemento sin sacarlo.
* **Cola de prioridad:** cola que saca primero el elemento con mejor prioridad, no el más antiguo.
* **Lista enlazada:** secuencia de nodos donde cada uno apunta al siguiente (y al anterior, si es doble).
* **Nodo:** cada eslabón de una lista enlazada: un valor más los enlaces a sus vecinos.

-----

## El problema

Una lista sirve para casi todo, pero a veces **cómo** se agregan y se sacan los elementos es parte de la regla del negocio:

* Una impresora atiende los documentos **en orden de llegada**. Si usas una lista y sacas del principio (`RemoveAt(0)`), cada vez desplazas todos los demás elementos, y nada impide que alguien saque uno del medio y "se cuele".
* Un editor de texto tiene "deshacer": la **última** acción es la primera que se revierte.
* Un sistema de soporte atiende primero los tickets **urgentes**, aunque hayan llegado después.

Usar una estructura específica hace que el código exprese la intención y que sea **imposible** romper la regla por accidente.

-----

## Cómo funciona

### `Queue<T>`: primero en entrar, primero en salir

```text
   Enqueue →  [ C ][ B ][ A ]  → Dequeue
              final       frente
```

```csharp
var impresora = new Queue<string>();

impresora.Enqueue("informe.pdf");     // al final
impresora.Enqueue("foto.png");
impresora.Enqueue("contrato.docx");

Console.WriteLine(impresora.Peek());      // informe.pdf (mira el frente sin sacarlo)
Console.WriteLine(impresora.Dequeue());   // informe.pdf (lo saca)
Console.WriteLine(impresora.Count);       // 2

while (impresora.Count > 0)
{
    Console.WriteLine($"Imprimiendo {impresora.Dequeue()}");
}
// foto.png, contrato.docx: en orden de llegada
```

`Dequeue` y `Peek` sobre una cola vacía lanzan `InvalidOperationException: Queue empty.` Las versiones `Try` evitan la excepción:

```csharp
if (impresora.TryDequeue(out string? doc))
{
    Console.WriteLine(doc);
}
```

Usos típicos: procesar pedidos o mensajes en orden, recorrer un árbol o un grafo "por niveles" (búsqueda en anchura), colas de trabajo.

### `Stack<T>`: último en entrar, primero en salir

```text
   Push ↓  ↑ Pop
        [ C ]   ← tope
        [ B ]
        [ A ]
```

```csharp
var historial = new Stack<string>();

historial.Push("Escribir 'Hola'");
historial.Push("Poner en negrita");
historial.Push("Cambiar color");

Console.WriteLine(historial.Peek());   // Cambiar color (el tope)
Console.WriteLine(historial.Pop());    // Cambiar color (deshacer la última acción)
Console.WriteLine(historial.Pop());    // Poner en negrita
Console.WriteLine(historial.Count);    // 1
```

Igual que la cola: `Pop` y `Peek` sobre una pila vacía lanzan `InvalidOperationException: Stack empty.`; usa `TryPop` y `TryPeek`.

Al recorrer una pila con `foreach`, los elementos salen desde el tope (el último que entró):

```csharp
var s = new Stack<int>();
s.Push(1); s.Push(2); s.Push(3);
Console.WriteLine(string.Join(", ", s));   // 3, 2, 1
```

Usos típicos: deshacer/rehacer, navegación "atrás" de un navegador, verificar paréntesis balanceados, evaluar expresiones, recorrer en profundidad.

### `PriorityQueue<TElement, TPriority>`: primero el más importante

```csharp
var tickets = new PriorityQueue<string, int>();   // menor número = mayor prioridad

tickets.Enqueue("Cambiar contraseña", 3);
tickets.Enqueue("Servidor caído", 1);
tickets.Enqueue("Error en factura", 2);

while (tickets.TryDequeue(out string? ticket, out int prioridad))
{
    Console.WriteLine($"[P{prioridad}] {ticket}");
}
// [P1] Servidor caído
// [P2] Error en factura
// [P3] Cambiar contraseña
```

* Sale primero el elemento con la **menor** prioridad (por defecto).
* Con prioridades iguales, el orden **no** está garantizado (no es estable).
* No se puede recorrer en orden de prioridad con `foreach`: solo sacando elementos.

### `LinkedList<T>`: una cadena de nodos

En una `List<T>`, los elementos están contiguos en un array; insertar en el medio obliga a desplazar todo lo que sigue. En una `LinkedList<T>`, cada elemento es un **nodo** que apunta al anterior y al siguiente:

```text
null ← [Lima] ⇄ [Cusco] ⇄ [Piura] → null
        First                Last
```

Insertar o quitar un nodo solo cambia los enlaces de sus vecinos:

```csharp
var ruta = new LinkedList<string>();

ruta.AddLast("Lima");                          // Lima
ruta.AddLast("Piura");                         // Lima ⇄ Piura
ruta.AddFirst("Tacna");                        // Tacna ⇄ Lima ⇄ Piura

LinkedListNode<string>? lima = ruta.Find("Lima");
if (lima is not null)
{
    ruta.AddAfter(lima, "Cusco");              // Tacna ⇄ Lima ⇄ Cusco ⇄ Piura
    Console.WriteLine(lima.Next!.Value);       // Cusco
    Console.WriteLine(lima.Previous!.Value);   // Tacna
}

ruta.RemoveFirst();                            // Lima ⇄ Cusco ⇄ Piura
ruta.Remove("Piura");                          // Lima ⇄ Cusco
Console.WriteLine(string.Join(" → ", ruta));   // Lima → Cusco
```

* `First` y `Last` dan los nodos de los extremos (o `null` si está vacía).
* Cada `LinkedListNode<T>` tiene `Value`, `Next` y `Previous`.
* `Find(valor)` recorre la lista (O(n)), pero, **una vez que tienes el nodo**, insertar o quitar a su lado es O(1).
* No tiene indexador: `ruta[2]` no existe. Para llegar al tercer elemento hay que recorrer.

En la práctica, `LinkedList<T>` se usa poco: `List<T>` suele ser más rápida incluso para inserciones, porque su memoria contigua aprovecha mejor la caché del procesador. Tiene sentido cuando guardas referencias a nodos y haces muchas inserciones y borrados en el medio (por ejemplo, una caché LRU).

-----

## Ejemplo completo

Un editor de texto mínimo con deshacer y rehacer (dos pilas) y una cola de guardado:

```csharp
var editor = new Editor();

editor.Escribir("Hola");
editor.Escribir(" mundo");
editor.Escribir("!");
Console.WriteLine(editor.Texto);            // Hola mundo!

editor.Deshacer();
editor.Deshacer();
Console.WriteLine(editor.Texto);            // Hola

editor.Rehacer();
Console.WriteLine(editor.Texto);            // Hola mundo

editor.Escribir(" cruel");                  // una acción nueva borra el historial de rehacer
Console.WriteLine(editor.Texto);            // Hola mundo cruel
Console.WriteLine($"¿Se puede rehacer? {editor.PuedeRehacer}");   // False

editor.ProcesarGuardados();

class Editor
{
    private readonly Stack<string> _deshacer = new();
    private readonly Stack<string> _rehacer = new();
    private readonly Queue<string> _pendientesDeGuardar = new();

    public string Texto { get; private set; } = "";
    public bool PuedeRehacer => _rehacer.Count > 0;

    public void Escribir(string fragmento)
    {
        _deshacer.Push(Texto);          // guardo el estado ANTERIOR
        _rehacer.Clear();
        Texto += fragmento;
        _pendientesDeGuardar.Enqueue(Texto);
    }

    public void Deshacer()
    {
        if (!_deshacer.TryPop(out string? anterior)) return;
        _rehacer.Push(Texto);
        Texto = anterior;
    }

    public void Rehacer()
    {
        if (!_rehacer.TryPop(out string? siguiente)) return;
        _deshacer.Push(Texto);
        Texto = siguiente;
    }

    public void ProcesarGuardados()
    {
        int n = 1;
        while (_pendientesDeGuardar.TryDequeue(out string? version))
        {
            Console.WriteLine($"Guardando versión {n++}: \"{version}\"");
        }
    }
}
```

Salida:

```text
Hola mundo!
Hola
Hola mundo
Hola mundo cruel
¿Se puede rehacer? False
Guardando versión 1: "Hola"
Guardando versión 2: "Hola mundo"
Guardando versión 3: "Hola mundo!"
Guardando versión 4: "Hola mundo cruel"
```

Las pilas implementan el deshacer y el rehacer; la cola guarda las versiones **en el orden en que se produjeron**.

-----

## Errores comunes

**1. Sacar de una cola o pila vacía.**
Qué pasa: `System.InvalidOperationException: Queue empty.` (o `Stack empty.`).
Por qué: `Dequeue`, `Pop` y `Peek` exigen al menos un elemento.
Arreglo: `TryDequeue`, `TryPop`, `TryPeek`, o comprobar `Count > 0`.

**2. Buscar un método `IsEmpty`.**
Qué pasa: `error CS1061: 'Queue<string>' does not contain a definition for 'IsEmpty'`.
Por qué: no existe en estas colecciones.
Arreglo: `cola.Count == 0`.

**3. Esperar que `foreach` sobre una pila recorra en orden de inserción.**
Qué pasa: los elementos salen al revés.
Por qué: la pila se recorre desde el tope (LIFO).
Arreglo: si necesitas el orden de inserción, usa `s.Reverse()` (LINQ) o elige otra estructura.

**4. Esperar orden estable en `PriorityQueue` con prioridades iguales.**
Qué pasa: dos tickets con la misma prioridad salen en un orden distinto al de llegada.
Por qué: la cola de prioridad no es estable.
Arreglo: usa una prioridad compuesta, por ejemplo una tupla `(prioridad, númeroDeLlegada)`.

**5. Usar índices en una `LinkedList<T>`.**
Qué pasa: `error CS0021: Cannot apply indexing with [] to an expression of type 'LinkedList<string>'`.
Por qué: una lista enlazada no tiene acceso por posición.
Arreglo: recorre los nodos, o usa `List<T>` si necesitas índices.

-----

## Según la versión de C#

* **.NET 2.0:** `Queue<T>`, `Stack<T>` y `LinkedList<T>` genéricas (antes existían `Queue` y `Stack` no genéricas, con `object`).
* **.NET Core 2.0:** `TryDequeue`, `TryPeek` y `TryPop`.
* **.NET 6:** `PriorityQueue<TElement, TPriority>`.
* **.NET 7+:** `PriorityQueue` agrega `DequeueEnqueue` y, en versiones recientes, `Remove` de un elemento concreto.

-----

## Cuándo sí y cuándo no

| Si necesitas... | Usa |
| --- | --- |
| Procesar en orden de llegada | `Queue<T>` |
| Deshacer, volver atrás, lo último primero | `Stack<T>` |
| Atender primero lo más importante | `PriorityQueue<T, P>` |
| Muchas inserciones y borrados en el medio con referencias a los nodos | `LinkedList<T>` |
| Acceso por índice y uso general | `List<T>` |
| Cola compartida entre hilos | `ConcurrentQueue<T>` o `Channel<T>` |

-----

## Resumen en 5 líneas

1. `Queue<T>`: `Enqueue` al final y `Dequeue` del frente (FIFO).
2. `Stack<T>`: `Push` y `Pop` en el tope (LIFO); su `foreach` recorre desde el último.
3. `PriorityQueue<T, P>`: sale primero la menor prioridad; no es estable con empates.
4. `Peek` mira sin sacar; con la colección vacía, `Dequeue`/`Pop`/`Peek` lanzan excepción: usa las versiones `Try`.
5. `LinkedList<T>`: nodos con `Next`/`Previous`; inserción O(1) junto a un nodo conocido, sin acceso por índice.

-----

## Para profundizar

<details>
<summary>Verificar paréntesis balanceados con una pila</summary>

Un ejercicio clásico de entrevista:

```csharp
static bool Balanceado(string expresion)
{
    var pares = new Dictionary<char, char> { [')'] = '(', [']'] = '[', ['}'] = '{' };
    var pila = new Stack<char>();

    foreach (char c in expresion)
    {
        if (c is '(' or '[' or '{') pila.Push(c);
        else if (pares.TryGetValue(c, out char apertura))
        {
            if (!pila.TryPop(out char ultimo) || ultimo != apertura) return false;
        }
    }
    return pila.Count == 0;
}

Console.WriteLine(Balanceado("{[()()]}"));   // True
Console.WriteLine(Balanceado("([)]"));       // False
```

Cada cierre debe coincidir con la **última** apertura pendiente: exactamente lo que modela una pila.

</details>

<details>
<summary>Cómo están implementadas</summary>

`Queue<T>` usa un array circular: dos índices (cabeza y cola) que avanzan y "dan la vuelta", así que `Enqueue` y `Dequeue` son O(1) sin desplazar elementos. `Stack<T>` es un array donde se agrega y se quita siempre al final. `PriorityQueue` es un *heap* binario (en realidad cuaternario en .NET): `Enqueue` y `Dequeue` son O(log n). `LinkedList<T>` es una lista doblemente enlazada circular.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una `Queue<T>` es FIFO: el primero en entrar es el primero en salir, con `Enqueue` y `Dequeue`. Una `Stack<T>` es LIFO: el último en entrar es el primero en salir, con `Push` y `Pop`. Ambas tienen `Peek` para mirar sin sacar. Una `LinkedList<T>` es una lista doblemente enlazada, útil para insertar o quitar en el medio.

### Respuesta ampliada (semi-senior)

`Queue<T>` es un buffer circular con `Enqueue`/`Dequeue` O(1); `Stack<T>`, un array con `Push`/`Pop` O(1) amortizado; ambas lanzan `InvalidOperationException` si están vacías, por lo que se prefieren las variantes `Try`. `PriorityQueue<TElement, TPriority>` (.NET 6) es un heap con O(log n) y no es estable ante empates. `LinkedList<T>` ofrece O(1) para insertar o quitar dado un nodo, pero carece de indexador y tiene peor localidad de caché que `List<T>`, por lo que solo conviene si se mantienen referencias a los nodos. Para escenarios con varios hilos se usan `ConcurrentQueue`/`ConcurrentStack` o `System.Threading.Channels`.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `Peek` y `Dequeue`?**
`Peek` devuelve el siguiente elemento sin sacarlo; `Dequeue` lo devuelve y lo quita.

**2. ¿Cómo implementarías "deshacer y rehacer"?**
Con dos pilas: al deshacer, se saca de la pila de deshacer y se apila en la de rehacer; una acción nueva vacía la pila de rehacer.

**3. ¿Cuándo usarías `LinkedList<T>` en lugar de `List<T>`?**
Cuando hay muchas inserciones y borrados en posiciones intermedias y ya tienes referencias a los nodos (por ejemplo, una caché LRU). En otros casos, `List<T>` suele ser más rápida.

-----

## Práctica

**Ejercicio 1.** Simula una fila de banco: encola a 4 clientes, atiende (desencola) a 2 mostrando sus nombres, y luego muestra quién es el siguiente sin sacarlo y cuántos quedan.

<details>
<summary>Solución</summary>

```csharp
var fila = new Queue<string>(new[] { "Ana", "Luis", "Eva", "Juan" });

Console.WriteLine($"Atendiendo a {fila.Dequeue()}");   // Ana
Console.WriteLine($"Atendiendo a {fila.Dequeue()}");   // Luis
Console.WriteLine($"Siguiente: {fila.Peek()}");         // Eva
Console.WriteLine($"Quedan: {fila.Count}");             // 2
```

</details>

**Ejercicio 2.** Usa una `Stack<char>` para invertir una palabra y comprobar si es un palíndromo (se lee igual al revés), ignorando mayúsculas.

<details>
<summary>Solución</summary>

```csharp
Console.WriteLine(EsPalindromo("Reconocer"));   // True
Console.WriteLine(EsPalindromo("Hola"));        // False

static bool EsPalindromo(string palabra)
{
    string normal = palabra.ToLowerInvariant();
    var pila = new Stack<char>(normal);         // apila cada carácter
    string invertida = new string(pila.ToArray());   // ToArray recorre desde el tope: queda invertida
    return normal == invertida;
}
```

</details>

-----

## Siguiente lección

[Elegir la colección adecuada](04-Elegir%20la%20coleccion%20adecuada.md)
