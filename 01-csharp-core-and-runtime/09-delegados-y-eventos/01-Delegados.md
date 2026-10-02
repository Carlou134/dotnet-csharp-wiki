# Delegados

## En una frase

Un **delegado** es un tipo cuyas variables guardan **referencias a métodos** con una firma concreta, para pasar comportamiento como si fuera un dato, cambiarlo en tiempo de ejecución y encadenar varios métodos en una sola invocación (**multicast**).

-----

## Antes de empezar

Conviene que ya sepas:

* Lambdas, `Func`, `Action` y `Predicate`, y pasar métodos como argumentos, de [Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md).
* Métodos estáticos y de instancia, de [Miembros estáticos](../04-poo/05-Miembros%20estaticos.md).
* Genéricos, de [Genéricos](../05-tipos-avanzados/04-Genericos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Delegado:** tipo que representa métodos con una firma determinada (parámetros y tipo de retorno).
* **Grupo de métodos (*method group*):** el nombre de un método sin paréntesis, usado como valor (`MiMetodo`).
* **Invocar:** ejecutar el método al que apunta un delegado: `delegado(args)` o `delegado.Invoke(args)`.
* **Delegado multicast:** delegado que apunta a varios métodos y los ejecuta todos, en orden.
* **Lista de invocación:** la lista de métodos de un delegado multicast.
* **Método anónimo:** método sin nombre escrito con `delegate (...) { }` (la forma anterior a las lambdas).
* **Callback:** método que pasas a otro para que lo llame cuando corresponda.

-----

## El problema

Escribes una función que procesa texto, pero el procesamiento cambia según el caso: a veces hay que pasar a mayúsculas, a veces agregar un punto, a veces ambas. Sin delegados:

```csharp
string Procesar(string texto, string modo)
{
    if (modo == "mayusculas") return texto.ToUpper();
    if (modo == "punto") return texto + ".";
    if (modo == "ambos") return texto.ToUpper() + ".";
    throw new ArgumentException("Modo desconocido");
}
```

Cada modo nuevo obliga a modificar esta función, los modos son strings que se pueden escribir mal, y quien la usa no puede agregar su propio procesamiento. Lo que quieres es que quien llama **entregue el comportamiento**: "procesa este texto **con esta función**".

-----

## Cómo funciona

### Declarar un tipo delegado

```csharp
delegate string ProcesadorDeTexto(string texto);
```

Se lee: "`ProcesadorDeTexto` es el tipo de cualquier método que recibe un `string` y devuelve un `string`". La declaración se parece a la firma de un método, con `delegate` delante.

Un delegado es un **tipo**, como una clase: se puede declarar en un espacio de nombres, después de las top-level statements o **anidado dentro de una clase** (`public delegate void Manejador(int x);` dentro de `class Publicador`).

### Asignar métodos y llamarlos

```csharp
ProcesadorDeTexto proceso = Mayusculas;     // grupo de métodos: SIN paréntesis
string r1 = proceso("hola");                // invocar: CON paréntesis → "HOLA"

proceso = AgregarPunto;                     // cambiar el comportamiento en ejecución
string r2 = proceso("hola");                // "hola."

string r3 = proceso.Invoke("hola");         // forma explícita, equivalente

static string Mayusculas(string t) => t.ToUpper();
static string AgregarPunto(string t) => t + ".";
```

* Sin paréntesis (`Mayusculas`) nombras el método; con paréntesis lo **ejecutas**.
* El método debe coincidir con la firma del delegado: mismos tipos de parámetros y de retorno.

### Delegados genéricos de .NET

En la práctica casi nunca declaras tus propios delegados: .NET trae delegados genéricos que cubren casi todas las firmas.

```csharp
Func<string, string> proceso = Mayusculas;     // equivale a ProcesadorDeTexto
Func<double, int> redondear = d => (int)Math.Round(d);
Func<int, double, bool> esMayor = (a, b) => a > b;   // el ÚLTIMO tipo es el retorno

Action<string> imprimir = Console.WriteLine;   // sin retorno
Action limpiar = Console.Clear;

Predicate<int> esPar = n => n % 2 == 0;        // recibe T, devuelve bool
```

Regla para leer un `Func`: **todos los tipos son parámetros, salvo el último, que es el retorno**. Un método `string Combinar(string a, int b)` es un `Func<string, int, string>`.

¿Cuándo declarar tu propio delegado?

* Cuando el nombre aporta significado (`ValidadorDePedido` dice más que `Func<Pedido, bool>`).
* Cuando necesitas parámetros `ref`, `out` o `params`, que `Func` y `Action` no admiten.

### De dónde pueden salir los métodos

```csharp
// 1. Métodos estáticos (propios o de .NET)
Func<double, double> raiz = Math.Sqrt;
Action<string> escribir = Console.WriteLine;

// 2. Métodos de INSTANCIA: el delegado guarda también el objeto
var parrafo = new Parrafo("Empezaron a caminar");
Func<string, string> agregar = parrafo.AgregarOracion;
agregar("Siguieron caminando");
Console.WriteLine(parrafo.Contenido);   // Empezaron a caminar. Siguieron caminando

// 3. Lambdas
Func<string, string> duplicar = s => s + s;

class Parrafo
{
    public string Contenido { get; private set; }
    public Parrafo(string inicio) => Contenido = inicio;
    public string AgregarOracion(string siguiente) => Contenido += ". " + siguiente;
}
```

Un delegado a un método de instancia guarda **el método y el objeto** sobre el que se llama: invocarlo modifica ese objeto concreto.

Un error común: `Func<string, string> f = string.ToUpper;` **no compila**. `ToUpper` es un método de **instancia** sin parámetros (se llama sobre un string), no un método estático que reciba un string. Lo correcto es una lambda: `Func<string, string> f = s => s.ToUpper();`.

La visibilidad también cuenta: solo puedes asignar métodos accesibles desde donde estás (un método `private` solo dentro de su clase).

### Métodos anónimos (sintaxis antigua)

Antes de las lambdas (C# 2), se escribían así:

```csharp
Func<int, int> doble = delegate (int x) { return x * 2; };
```

Hoy se usa la lambda equivalente `x => x * 2`. Reconócelos en código antiguo.

### Delegados multicast

Un delegado puede apuntar a **varios** métodos. Con `+=` se agregan a la lista de invocación y con `-=` se quitan:

```csharp
Action<string> log = MensajeConsola;
log += MensajeArchivo;
log += m => Console.WriteLine($"[auditoría] {m}");

log("Usuario conectado");   // ejecuta los tres, en el orden en que se agregaron

log -= MensajeConsola;      // quita uno
log("Usuario desconectado");  // ejecuta los dos restantes

static void MensajeConsola(string m) => Console.WriteLine($"[consola] {m}");
static void MensajeArchivo(string m) => File.AppendAllText("log.txt", m + Environment.NewLine);
```

```text
log  ──►  lista de invocación
          ┌─────────────────┬─────────────────┬──────────────────┐
          │ MensajeConsola  │ MensajeArchivo  │ lambda auditoría │
          └─────────────────┴─────────────────┴──────────────────┘
               1º                 2º                 3º             ← log("...") los llama en este orden

log -= MensajeConsola   →   ┌─────────────────┬──────────────────┐
                            │ MensajeArchivo  │ lambda auditoría │   (un delegado NUEVO: son inmutables)
                            └─────────────────┴──────────────────┘
```

Detalles importantes:

* Si quitas todos los métodos, el delegado queda en **`null`**; invocarlo lanza `NullReferenceException`. Por eso se invoca con `delegado?.Invoke(args)`.
* Si el delegado **devuelve un valor**, la invocación devuelve el del **último** método; los demás resultados se descartan (no se "encadenan").
* Si un método de la lista lanza una excepción, los siguientes **no se ejecutan**.
* `-=` sobre una lambda escrita de nuevo no funciona: cada lambda es un objeto distinto. Para poder quitarla, guárdala en una variable.

```csharp
Func<int, int, int> op = (a, b) => a + b;
op += (a, b) => a - b;
Console.WriteLine(op(5, 3));   // 2: solo cuenta el resultado del último (5 - 3)
```

### `Predicate<T>` y los métodos de `List<T>`

`List<T>` tiene métodos que reciben un `Predicate<T>`:

```csharp
var numeros = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
Predicate<int> esPar = n => n % 2 == 0;

List<int> pares = numeros.FindAll(esPar);        // [2, 4, 6, 8, 10]
int primerPar = numeros.Find(esPar);             // 2
bool hayPar = numeros.Exists(esPar);             // true
bool todosPares = numeros.TrueForAll(esPar);     // false
int quitados = numeros.RemoveAll(esPar);         // 5
```

(Estos son métodos de `List<T>`, no de LINQ; LINQ usa `Func<T, bool>`. Hacen lo mismo con otro tipo de delegado).

### Delegados como estrategia

Pasar un delegado a un método permite cambiar una parte de su comportamiento sin modificarlo (el patrón Strategy en versión liviana):

```csharp
decimal Total(IEnumerable<decimal> precios, Func<decimal, decimal> aplicarDescuento)
{
    decimal total = 0;
    foreach (var p in precios) total += aplicarDescuento(p);
    return total;
}

var precios = new[] { 100m, 250m, 40m };
Console.WriteLine(Total(precios, p => p));                         // 390 sin descuento
Console.WriteLine(Total(precios, p => p * 0.9m));                  // 351 con 10 %
Console.WriteLine(Total(precios, p => p > 50 ? p - 20 : p));       // 350 con 20 de descuento en los caros
```

-----

## Ejemplo completo

Un pipeline de procesamiento de texto configurable y un sistema de notificaciones multicast:

```csharp
// 1. Pipeline: lista de transformaciones que se aplican en orden
var pasos = new List<Func<string, string>>
{
    s => s.Trim(),
    s => s.ToLowerInvariant(),
    QuitarDobleEspacio,
    s => char.ToUpper(s[0]) + s[1..],
    s => s.EndsWith('.') ? s : s + "."
};

string entrada = "   HOLA   MUNDO   desde   C#  ";
string resultado = pasos.Aggregate(entrada, (texto, paso) => paso(texto));
Console.WriteLine($"[{resultado}]");

// 2. Validaciones como predicados con nombre
var validaciones = new Dictionary<string, Predicate<string>>
{
    ["no está vacío"] = s => !string.IsNullOrWhiteSpace(s),
    ["termina en punto"] = s => s.EndsWith('.'),
    ["menos de 30 caracteres"] = s => s.Length < 30
};

foreach (var (nombre, regla) in validaciones)
    Console.WriteLine($"{(regla(resultado) ? "✔" : "✘")} {nombre}");

// 3. Notificador multicast
Notificar notificar = Correo;
notificar += Sms;
notificar += (dest, msg) => Console.WriteLine($"  [push] {dest}: {msg}");

Console.WriteLine($"Canales: {notificar.GetInvocationList().Length}");
notificar("ana", "Tu pedido salió");

notificar -= Sms;
Console.WriteLine($"Canales después de quitar SMS: {notificar.GetInvocationList().Length}");
notificar("ana", "Tu pedido llegó");

static string QuitarDobleEspacio(string s)
{
    while (s.Contains("  ")) s = s.Replace("  ", " ");
    return s;
}

static void Correo(string destino, string mensaje) => Console.WriteLine($"  [correo] {destino}: {mensaje}");
static void Sms(string destino, string mensaje) => Console.WriteLine($"  [sms] {destino}: {mensaje}");

delegate void Notificar(string destino, string mensaje);
```

Salida:

```text
[Hola mundo desde c#.]
✔ no está vacío
✔ termina en punto
✔ menos de 30 caracteres
Canales: 3
  [correo] ana: Tu pedido salió
  [sms] ana: Tu pedido salió
  [push] ana: Tu pedido salió
Canales después de quitar SMS: 2
  [correo] ana: Tu pedido llegó
  [push] ana: Tu pedido llegó
```

Agregar un paso al pipeline o una regla de validación es agregar una línea, sin tocar el código que los ejecuta.

-----

## Errores comunes

**1. Poner paréntesis al asignar.**
Qué pasa: `proceso = Mayusculas();` da `error CS7036: There is no argument given that corresponds to the required parameter 't'`.
Por qué: con paréntesis estás llamando al método.
Arreglo: `proceso = Mayusculas;`.

**2. Firma incompatible.**
Qué pasa: `error CS0123: No overload for 'Mayusculas' matches delegate 'Func<int, string>'`.
Por qué: el método no recibe o no devuelve los tipos del delegado.
Arreglo: ajusta el delegado o el método, o adapta con una lambda.

**3. Asignar un método de instancia como si fuera estático.**
Qué pasa: `Func<string, string> f = string.ToUpper;` da un error de compilación (no hay sobrecarga de `ToUpper` que reciba un `string`).
Por qué: `ToUpper` se llama sobre un string; no recibe el string como parámetro.
Arreglo: `s => s.ToUpper()`.

**4. Invocar un delegado `null`.**
Qué pasa: `NullReferenceException` al quitar el último método y luego invocar.
Por qué: un delegado sin métodos es `null`.
Arreglo: `delegado?.Invoke(args)`.

**5. Esperar todos los resultados de un multicast con retorno.**
Qué pasa: solo obtienes el resultado del último método.
Por qué: así funciona la invocación multicast.
Arreglo: recorre `GetInvocationList()` y llama a cada uno, o usa una lista de `Func`.

**6. Quitar una lambda con `-=` escribiéndola de nuevo.**
Qué pasa: no se quita nada (y no hay error).
Por qué: dos lambdas iguales son objetos distintos.
Arreglo: guarda la lambda en una variable y usa esa variable en `+=` y `-=`.

-----

## Según la versión de C#

* **C# 1:** tipos delegados, `+=` y `-=` (con `new MiDelegado(Metodo)` explícito).
* **C# 2:** métodos anónimos (`delegate { }`) y conversión de grupo de métodos (ya no hace falta `new MiDelegado(...)`).
* **C# 3 / .NET 3.5:** lambdas y los delegados genéricos `Func` y `Action`.
* **C# 9:** lambdas `static` y descartes como parámetros.
* **C# 10:** tipo natural de lambdas y grupos de métodos: `var f = Math.Sqrt;` (si hay una sola sobrecarga).

-----

## Cuándo sí y cuándo no

**Usa delegados cuando:**

* Una parte del comportamiento la decide quien llama: filtros, transformaciones, estrategias, callbacks.
* Quieres configurar una secuencia de pasos o reglas como datos.

**Prefiere una interfaz cuando:**

* El comportamiento tiene varios métodos relacionados o estado propio: una interfaz con una clase que la implementa es más clara que varios delegados sueltos.

**Para notificar a varios interesados:**

* Usa **eventos** (próxima lección), que son delegados multicast con protección extra.

-----

## Resumen en 5 líneas

1. Un delegado es un tipo para referencias a métodos con una firma: `delegate string Proceso(string s);`.
2. Se asigna un método sin paréntesis y se invoca con paréntesis o `Invoke`; puede ser estático, de instancia o una lambda.
3. `Func<..., TResult>`, `Action<...>` y `Predicate<T>` cubren casi todas las firmas: el último tipo de `Func` es el retorno.
4. Multicast: `+=` agrega y `-=` quita; se ejecutan en orden y, con retorno, solo vale el del último.
5. Un delegado sin métodos es `null`: invócalo con `?.Invoke`.

-----

## Para profundizar

<details>
<summary>Qué es un delegado por dentro</summary>

Un tipo delegado es una clase que deriva de `System.MulticastDelegate`. Cada instancia guarda un puntero al método y, si es de instancia, una referencia al objeto (`Target`). Los delegados son **inmutables**: `a += b` crea un delegado nuevo con la lista combinada, igual que `+` con strings. Por eso es seguro copiar un delegado a una variable local antes de invocarlo, aunque otro hilo lo modifique al mismo tiempo.

</details>

<details>
<summary>Callbacks y el costo de los cierres</summary>

Si la lambda captura variables (una clausura), cada vez que se crea genera un objeto en el heap. En código muy caliente (millones de llamadas), conviene usar lambdas `static` (que no capturan) o pasar el estado como parámetro. Para el resto del código, la claridad vale más que esa diferencia.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un delegado es un tipo que permite guardar referencias a métodos en variables, pasarlos como parámetros y ejecutarlos después. .NET trae delegados genéricos: `Func` (devuelve un valor), `Action` (no devuelve nada) y `Predicate` (devuelve `bool`). Un delegado puede apuntar a varios métodos con `+=`, y se ejecutan todos en orden.

### Respuesta ampliada (semi-senior)

Los delegados son tipos de referencia inmutables que derivan de `MulticastDelegate` y encapsulan un método y, opcionalmente, su `Target`. Admiten conversión de grupos de métodos, métodos anónimos y lambdas, y son variantes (`Func<in T, out TResult>`). En la invocación multicast, el valor de retorno es el del último delegado y una excepción corta la cadena; `GetInvocationList()` permite invocar cada uno por separado. Son la base de los eventos, de LINQ y de los callbacks, y una forma liviana del patrón Strategy. Las lambdas que capturan variables generan clausuras en el heap.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `Func` y `Action`?**
`Func` devuelve un valor (el último parámetro de tipo); `Action` devuelve `void`.

**2. ¿Qué es un delegado multicast?**
Un delegado con varios métodos en su lista de invocación, que se ejecutan en orden al invocarlo.

**3. ¿Qué devuelve un delegado multicast con tipo de retorno?**
El valor del último método invocado; los anteriores se descartan.

-----

## Práctica

**Ejercicio 1.** Escribe un método `Aplicar(int[] numeros, Func<int, int> transformacion)` que devuelva un array nuevo con la transformación aplicada. Úsalo con un método con nombre (`Cuadrado`) y con dos lambdas distintas.

<details>
<summary>Solución</summary>

```csharp
int[] datos = { 1, 2, 3, 4 };

Console.WriteLine(string.Join(", ", Aplicar(datos, Cuadrado)));       // 1, 4, 9, 16
Console.WriteLine(string.Join(", ", Aplicar(datos, n => n + 10)));    // 11, 12, 13, 14
Console.WriteLine(string.Join(", ", Aplicar(datos, n => -n)));        // -1, -2, -3, -4

static int[] Aplicar(int[] numeros, Func<int, int> transformacion)
{
    var resultado = new int[numeros.Length];
    for (int i = 0; i < numeros.Length; i++)
        resultado[i] = transformacion(numeros[i]);
    return resultado;
}

static int Cuadrado(int n) => n * n;
```

</details>

**Ejercicio 2.** ¿Qué imprime este código?

```csharp
Func<int, int> f = x => x + 1;
f += x => x * 10;
f += x => x - 3;

Action<string> a = s => Console.Write("A");
a += s => Console.Write("B");
a -= s => Console.Write("B");

Console.WriteLine(f(5));
a("x");
```

<details>
<summary>Solución</summary>

```text
2
AB
```

`f(5)` ejecuta las tres lambdas, pero devuelve solo el resultado de la última: `5 - 3 = 2`. El `-=` no quita nada porque la lambda que se quita es un objeto distinto de la que se agregó, así que `a` sigue imprimiendo "A" y "B".

</details>

-----

## Siguiente lección

[Eventos](02-Eventos.md)
