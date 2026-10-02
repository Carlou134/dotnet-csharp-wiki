# Expresiones lambda

## En una frase

Con `=>` puedes escribir métodos de una sola expresión en una línea (**cuerpo de expresión**) y crear **lambdas**, que son métodos anónimos que se pasan como argumento a otros métodos y se guardan en variables de tipo `Func`, `Action` o `Predicate`.

-----

## Antes de empezar

Conviene que ya sepas:

* Definir métodos con parámetros y valor de retorno, de [Valores de retorno y parámetros out](03-Valores%20de%20retorno%20y%20parametros%20out.md).
* Usar `Array.Find` y `Array.Exists`, de [Arrays](../02-control-de-flujo/03-Arrays.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Miembro con cuerpo de expresión (*expression-bodied*):** método cuyo cuerpo es una sola expresión, escrito con `=>`.
* **Expresión lambda:** método anónimo (sin nombre) escrito en el lugar donde se usa: `x => x * 2`.
* **Método anónimo:** método que no tiene nombre.
* **Delegado:** tipo que representa "un método con cierta firma". Permite guardar métodos en variables y pasarlos como argumento.
* **`Func<...>`:** delegado para métodos que **devuelven** un valor.
* **`Action<...>`:** delegado para métodos que **no devuelven** nada.
* **`Predicate<T>`:** delegado para métodos que reciben un `T` y devuelven `bool`.
* **Función de orden superior:** método que recibe o devuelve otro método.
* **Captura (clausura):** cuando una lambda usa variables del método donde fue creada.

-----

## El problema

Mira este método:

```csharp
static bool EsPar(int numero)
{
    return numero % 2 == 0;
}
```

Cuatro líneas, llaves y `return` para una sola expresión. Y si solo lo necesitas una vez, para preguntar "¿hay algún número par en este array?", tienes que salir a definirlo con nombre en otro lugar del archivo, y quien lea el código tiene que ir a buscarlo.

C# ofrece dos atajos con la misma flecha `=>`:

1. Escribir métodos cortos en una línea.
2. Escribir el método **directamente donde se usa**, sin nombre.

-----

## Cómo funciona

### Métodos con cuerpo de expresión

Si un método solo devuelve una expresión, quita las llaves y el `return`, y usa `=>`:

```csharp
// Forma clásica
static bool EsPar(int numero)
{
    return numero % 2 == 0;
}

// Con cuerpo de expresión: equivalente
static bool EsPar2(int numero) => numero % 2 == 0;
```

También sirve para métodos `void`:

```csharp
static void Gritar(string texto) => Console.WriteLine(texto.ToUpper());
```

La condición es que el cuerpo sea **una sola expresión**. Si necesitas varias sentencias, vuelve a la forma con llaves. La flecha `=>` a veces se llama "flecha gruesa" (*fat arrow*).

### Métodos como argumentos

Antes de las lambdas, hay que entender una idea: **un método se puede pasar como argumento a otro método**.

```csharp
int[] numeros = { 1, 3, 5, 6, 7, 8 };

bool hayPar = Array.Exists(numeros, EsPar);   // se pasa el método, SIN paréntesis
Console.WriteLine(hayPar);                     // True

static bool EsPar(int n) => n % 2 == 0;
```

* `EsPar` sin paréntesis es **el método mismo**, no una llamada.
* `Array.Exists` lo llama por dentro con cada elemento: `EsPar(1)`, `EsPar(3)`, `EsPar(5)`, `EsPar(6)`... y en cuanto uno devuelve `true`, devuelve `true`.

Esto funciona porque `Array.Exists` declara que su segundo parámetro es un `Predicate<int>`: "cualquier método que reciba un `int` y devuelva `bool`". `EsPar` cumple esa firma.

### Expresiones lambda

Una lambda es ese mismo método, pero escrito **en el lugar de uso y sin nombre**:

```csharp
bool hayPar = Array.Exists(numeros, (int n) => n % 2 == 0);
```

```text
(int n)       =>        n % 2 == 0
parámetros   "va a"     expresión que se devuelve
```

Se lee: "dado un `n`, devuelve `n % 2 == 0`".

### Formas cortas

El compilador puede **inferir** el tipo de los parámetros a partir de dónde se usa la lambda (`Array.Exists` sobre un `int[]` espera un `int`), así que puedes omitirlo:

```csharp
Array.Exists(numeros, (int n) => n % 2 == 0);   // tipo explícito
Array.Exists(numeros, (n) => n % 2 == 0);       // tipo inferido
Array.Exists(numeros, n => n % 2 == 0);         // un solo parámetro: sin paréntesis (lo más habitual)
```

Según la cantidad de parámetros:

```csharp
() => Console.WriteLine("Sin parámetros")     // sin parámetros: paréntesis vacíos obligatorios
x => x * 2                                     // uno: paréntesis opcionales
(a, b) => a + b                                // dos o más: paréntesis obligatorios
```

### Lambdas con varias sentencias

Si necesitas más de una sentencia, usa llaves y `return`:

```csharp
bool hayDocenaGrande = Array.Exists(numeros, n =>
{
    bool multiploDe12 = n % 12 == 0;
    bool mayorQue20 = n > 20;
    return multiploDe12 && mayorQue20;
});
```

| Forma | Sintaxis | Cuándo |
| --- | --- | --- |
| Lambda de expresión | `x => expresión` | Una sola expresión (la mayoría de los casos) |
| Lambda de sentencias | `x => { ...; return valor; }` | Varias sentencias |

### Guardar lambdas en variables: `Func`, `Action` y `Predicate`

Una lambda se puede guardar en una variable cuyo tipo es un **delegado**:

```csharp
Func<double, double> cuadrado = x => x * x;
Func<int, int, int> sumar = (a, b) => a + b;
Func<string> obtenerSaludo = () => "Hola";

Action<string> imprimir = texto => Console.WriteLine(texto);
Action limpiar = () => Console.Clear();

Predicate<int> esPositivo = n => n > 0;

Console.WriteLine(cuadrado(5));      // 25
Console.WriteLine(sumar(2, 3));      // 5
imprimir("Hola");                    // Hola
Console.WriteLine(esPositivo(-3));   // False
```

| Delegado | Significa | Ejemplo |
| --- | --- | --- |
| `Func<TResultado>` | Sin parámetros, devuelve `TResultado` | `Func<string>` |
| `Func<T, TResultado>` | Recibe un `T`, devuelve `TResultado` | `Func<int, string>` |
| `Func<T1, T2, TResultado>` | Recibe dos, devuelve uno | `Func<int, int, int>` |
| `Action` | Sin parámetros, no devuelve nada | `Action` |
| `Action<T>` | Recibe un `T`, no devuelve nada | `Action<string>` |
| `Predicate<T>` | Recibe un `T`, devuelve `bool` | `Predicate<int>` |

Regla para leer un `Func`: **el último tipo es siempre el de retorno**. `Func<int, int, string>` recibe dos `int` y devuelve un `string`.

### Escribir tus propias funciones de orden superior

Un método puede recibir una función como parámetro:

```csharp
Console.WriteLine(Aplicar(x => x * x, 4));     // 16
Console.WriteLine(Aplicar(x => x + 10, 5));    // 15
Console.WriteLine(Aplicar(Math.Sqrt, 9));      // 3  (un método existente también sirve)

static double Aplicar(Func<double, double> operacion, double valor)
{
    return operacion(valor);    // se llama como cualquier método
}
```

El método `Aplicar` no sabe **qué** operación va a ejecutar: la decide quien lo llama. Esa es la idea central: **pasar comportamiento como si fuera un dato**.

### Captura de variables

Una lambda puede usar variables del lugar donde se creó:

```csharp
int minimo = 5;
int[] numeros = { 2, 7, 4, 9 };

int[] grandes = Array.FindAll(numeros, n => n > minimo);   // usa 'minimo', que no es un parámetro
Console.WriteLine(string.Join(", ", grandes));             // 7, 9
```

La lambda **captura la variable**, no una copia de su valor. Si la variable cambia después, la lambda ve el cambio:

```csharp
int factor = 2;
Func<int, int> multiplicar = x => x * factor;

factor = 10;
Console.WriteLine(multiplicar(3));   // 30, no 6
```

### Lambdas que ya usaste (y que usarás mucho)

```csharp
Array.Find(alturas, h => h > 5);
Array.Sort(nombres, (a, b) => a.Length.CompareTo(b.Length));   // ordenar por largo
lista.Where(p => p.Activo).Select(p => p.Nombre);               // LINQ (lección futura)
```

-----

## Ejemplo completo

Un pequeño procesador de pedidos que recibe reglas como funciones:

```csharp
decimal[] montos = { 120m, 45.5m, 300m, 80m, 15m };

// Reglas guardadas en variables
Predicate<decimal> esGrande = m => m >= 100m;
Func<decimal, decimal> conImpuesto = m => m * 1.18m;
Action<string> log = mensaje => Console.WriteLine($"[LOG] {mensaje}");

// Usar las reglas con métodos de Array
decimal[] grandes = Array.FindAll(montos, esGrande);
log($"Pedidos grandes: {string.Join(", ", grandes)}");

// Pasar distintas funciones al mismo método
decimal totalSinImpuesto = Totalizar(montos, m => m);
decimal totalConImpuesto = Totalizar(montos, conImpuesto);
decimal totalConDescuento = Totalizar(montos, m => esGrande(m) ? m * 0.9m : m);

log($"Total sin impuesto: {totalSinImpuesto:N2}");
log($"Total con impuesto: {totalConImpuesto:N2}");
log($"Total con descuento en grandes: {totalConDescuento:N2}");

static decimal Totalizar(decimal[] valores, Func<decimal, decimal> transformar)
{
    decimal total = 0;
    foreach (decimal v in valores)
    {
        total += transformar(v);
    }
    return total;
}
```

Salida:

```text
[LOG] Pedidos grandes: 120, 300
[LOG] Total sin impuesto: 560.50
[LOG] Total con impuesto: 661.39
[LOG] Total con descuento en grandes: 518.50
```

`Totalizar` se escribió **una vez** y calcula tres totales distintos: el comportamiento cambia según la función que recibe.

-----

## Errores comunes

**1. Poner paréntesis al pasar un método.**
Qué pasa: `Array.Exists(numeros, EsPar())` da `error CS7036: There is no argument given that corresponds to the required parameter 'n'`.
Por qué: con paréntesis estás **llamando** al método, no pasándolo.
Arreglo: `Array.Exists(numeros, EsPar)`.

**2. Cuerpo de expresión con varias sentencias.**
Qué pasa: errores de sintaxis como `error CS1002: ; expected`.
Por qué: `=>` sin llaves admite una sola expresión.
Arreglo: usa llaves y `return`: `x => { var y = x * 2; return y + 1; }`.

**3. Olvidar `return` en una lambda con llaves.**
Qué pasa: `error CS1643: Not all code paths return a value in lambda expression of type 'Func<int, int>'`.
Por qué: con llaves, el `return` no es implícito.
Arreglo: agrega `return`.

**4. Firma incompatible con el delegado.**
Qué pasa: `Func<int, int> f = (a, b) => a + b;` da `error CS1593: Delegate 'Func<int, int>' does not take 2 arguments`.
Por qué: `Func<int, int>` recibe **un** parámetro (el segundo tipo es el retorno).
Arreglo: `Func<int, int, int>`.

**5. Asignar una lambda a `var` sin tipos (antes de C# 10).**
Qué pasa: `var f = x => x * 2;` da `error CS8917: The delegate type could not be inferred`.
Por qué: el compilador no sabe el tipo de `x`.
Arreglo: declara el tipo del delegado (`Func<int, int> f = x => x * 2;`) o el de los parámetros (`var f = (int x) => x * 2;`, desde C# 10).

**6. Sorprenderse por la captura.**
Qué pasa: una lambda usa un valor distinto del esperado porque la variable capturada cambió.
Por qué: se captura la variable, no su valor en ese momento.
Arreglo: copia el valor a una variable local que no cambie antes de crear la lambda.

-----

## Según la versión de C#

* **C# 2:** métodos anónimos con `delegate (int x) { return x * 2; }` (lo verás en código muy antiguo).
* **C# 3:** expresiones lambda (`x => x * 2`) y LINQ.
* **C# 6:** métodos y propiedades de solo lectura con cuerpo de expresión.
* **C# 7:** cuerpo de expresión también en constructores, accesores `get`/`set` y finalizadores.
* **C# 9:** lambdas `static` (no pueden capturar variables) y descartes como parámetros (`(_, _) => 0`).
* **C# 10:** tipo natural: `var f = (int x) => x * 2;` y atributos en lambdas.
* **C# 12:** valores por defecto en los parámetros de una lambda: `var saludar = (string n = "mundo") => $"Hola {n}";`.

-----

## Cuándo sí y cuándo no

**Usa cuerpo de expresión cuando:**

* El método es una sola expresión corta y clara.

**Usa una lambda cuando:**

* La lógica es corta y solo tiene sentido en ese lugar (filtros, ordenamientos, condiciones de búsqueda).

**Usa un método con nombre cuando:**

* La lógica tiene más de dos o tres líneas, se reutiliza o merece una prueba propia.
* El nombre aporta claridad: `Array.Exists(pedidos, EstaVencido)` se lee mejor que una lambda larga.

-----

## Resumen en 5 líneas

1. `static int Doble(int x) => x * 2;` es un método con cuerpo de expresión.
2. Un método se pasa como argumento sin paréntesis: `Array.Exists(numeros, EsPar)`.
3. Una lambda es un método anónimo en línea: `n => n % 2 == 0`; con llaves y `return` si tiene varias sentencias.
4. `Func<..., TResultado>` devuelve un valor (el último tipo), `Action<...>` no devuelve nada y `Predicate<T>` devuelve `bool`.
5. Las lambdas capturan variables externas (la variable, no una copia de su valor).

-----

## Para profundizar

<details>
<summary>Qué es realmente un delegado</summary>

`Func<int, bool>` es un **tipo delegado**: un tipo cuyos valores son referencias a métodos. Puedes declarar los tuyos:

```csharp
delegate bool Filtro(int valor);

Filtro esGrande = v => v > 100;
```

`Func`, `Action` y `Predicate` son delegados genéricos que trae .NET para no tener que declarar uno nuevo cada vez. Los delegados son la base de los **eventos**, que se estudian en [Delegados y eventos](../09-delegados-y-eventos/README.md).

</details>

<details>
<summary>Cómo implementa el compilador la captura</summary>

Cuando una lambda captura variables locales, el compilador genera una clase oculta (una clausura) con esas variables como campos, y la lambda se convierte en un método de esa clase. Las variables "locales" pasan a vivir en el heap, dentro de ese objeto. Por eso la lambda ve los cambios posteriores y por eso capturar en bucles calientes tiene un costo de memoria. Una lambda `static` garantiza que no captura nada.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una expresión lambda es una función anónima que se escribe con `=>`, por ejemplo `x => x * 2`. Se usa para pasar lógica como argumento a otros métodos, como en `Array.Find` o en LINQ. Se puede guardar en variables de tipo `Func` si devuelve algo, `Action` si no devuelve nada, o `Predicate` si devuelve un `bool`.

### Respuesta ampliada (semi-senior)

Las lambdas se convierten a un tipo delegado (`Func`, `Action`, delegados propios) o a un árbol de expresión (`Expression<Func<...>>`), que es lo que permite a Entity Framework traducir una lambda a SQL. Cuando capturan variables, el compilador genera una clase de clausura en el heap, y la captura es por variable, no por valor, lo que explica el clásico bug de capturar la variable de un `for`. Desde C# 9 se pueden marcar como `static` para impedir capturas accidentales; desde C# 10 tienen tipo natural, y desde C# 12 admiten parámetros con valores por defecto. Los miembros con cuerpo de expresión son solo azúcar sintáctico para miembros de una sola expresión.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `Func` y `Action`?**
`Func` devuelve un valor (el último parámetro de tipo es el retorno); `Action` devuelve `void`.

**2. ¿Qué es una clausura (*closure*)?**
Una función junto con las variables que captura de su entorno. En C#, el compilador las implementa con una clase oculta.

**3. ¿Qué diferencia hay entre una lambda que se pasa a `List.Where` y una que se pasa a `IQueryable.Where`?**
En la primera, la lambda se compila a un delegado y se ejecuta en memoria. En la segunda, se compila a un árbol de expresión que el proveedor (por ejemplo, EF Core) traduce a otro lenguaje, como SQL.

-----

## Práctica

**Ejercicio 1.** Reescribe este código usando cuerpo de expresión para los métodos y una lambda en lugar de `EsMultiploDe3`:

```csharp
int[] datos = { 4, 9, 10, 12, 7 };
int[] multiplos = Array.FindAll(datos, EsMultiploDe3);
Console.WriteLine(Formatear(multiplos));

static bool EsMultiploDe3(int n)
{
    return n % 3 == 0;
}

static string Formatear(int[] valores)
{
    return string.Join(" | ", valores);
}
```

<details>
<summary>Solución</summary>

```csharp
int[] datos = { 4, 9, 10, 12, 7 };
int[] multiplos = Array.FindAll(datos, n => n % 3 == 0);
Console.WriteLine(Formatear(multiplos));   // 9 | 12

static string Formatear(int[] valores) => string.Join(" | ", valores);
```

</details>

**Ejercicio 2.** Escribe un método `Repetir(int veces, Action<int> accion)` que ejecute `accion` pasándole el número de repetición (empezando en 1). Úsalo para imprimir `Vuelta 1`, `Vuelta 2` y `Vuelta 3`, y luego para sumar los números del 1 al 100 en una variable externa.

<details>
<summary>Solución</summary>

```csharp
Repetir(3, i => Console.WriteLine($"Vuelta {i}"));

int suma = 0;
Repetir(100, i => suma += i);      // la lambda captura y modifica 'suma'
Console.WriteLine(suma);            // 5050

static void Repetir(int veces, Action<int> accion)
{
    for (int i = 1; i <= veces; i++)
    {
        accion(i);
    }
}
```

</details>

-----

## Siguiente lección

Terminaste el módulo de métodos. Continúa con [Programación orientada a objetos](../04-poo/README.md).
