# Manejo de excepciones

## En una frase

Una **excepción** es un objeto que representa un error ocurrido durante la ejecución; si nadie la captura, se **propaga** hacia arriba por la pila de llamadas hasta detener el programa, y con `try`/`catch`/`finally` puedes **capturarla**, reaccionar y garantizar que el código de limpieza se ejecute siempre.

-----

## Antes de empezar

Conviene que ya sepas:

* Métodos y cómo se llaman unos a otros (la pila de llamadas), de [Definir y llamar métodos](../03-metodos/01-Definir%20y%20llamar%20metodos.md).
* Herencia, porque todas las excepciones heredan de `Exception`, de [Herencia](../04-poo/06-Herencia.md).
* `TryParse` como alternativa sin excepciones, de [Conversiones de tipos](../01-tipos-y-variables/03-Conversiones%20de%20tipos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Excepción:** objeto que describe un error en tiempo de ejecución; deriva de `System.Exception`.
* **Lanzar (*throw*):** producir una excepción.
* **Capturar (*catch*):** interceptar una excepción para manejarla.
* **Pila de llamadas (*call stack*):** la cadena de métodos que se llamaron hasta el punto actual.
* **Propagación:** el viaje de una excepción no capturada hacia el método que llamó, y así sucesivamente.
* **Traza de pila (*stack trace*):** el registro de esa cadena de llamadas, incluido en la excepción.
* **Filtro de excepción:** condición `when` que decide si un `catch` maneja la excepción.
* **Excepción no controlada:** la que llega al punto de entrada sin ser capturada y termina el proceso.

-----

## El problema

La ley de Murphy aplica al software: todo lo que pueda salir mal, saldrá mal. El usuario escribe letras donde va un número, el archivo no existe, la red se cae, el divisor es cero:

```csharp
Console.Write("Número: ");
int n = int.Parse(Console.ReadLine()!);   // el usuario escribe "hola"
Console.WriteLine(100 / n);
```

```text
Unhandled exception. System.FormatException: The input string 'hola' was not in a correct format.
   at System.Number.ThrowFormatException...
   at Program.<Main>$(String[] args) in Program.cs:line 2
```

El programa se cierra de golpe, sin un mensaje útil para el usuario y sin cerrar lo que tenía abierto. Necesitas una forma de **anticipar** esos fallos y decidir qué hacer: reintentar, mostrar un mensaje claro, registrar el error o cerrar ordenadamente.

-----

## Cómo funciona

### `try` y `catch`

```csharp
try
{
    int numerador = 10;
    int denominador = 0;
    int resultado = numerador / denominador;   // lanza DivideByZeroException
    Console.WriteLine(resultado);              // no se ejecuta
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"No se puede dividir por cero: {ex.Message}");
}

Console.WriteLine("El programa sigue");
```

1. Se ejecuta el bloque `try`.
2. Si una línea lanza una excepción, el resto del `try` **se salta** y la ejecución salta al `catch` cuyo tipo coincida.
3. Después del `catch`, el programa continúa normalmente.
4. Si no hubo excepción, el `catch` no se ejecuta.

`ex` es el objeto de la excepción. Sus miembros más útiles:

| Miembro | Contiene |
| --- | --- |
| `Message` | Descripción del error |
| `StackTrace` | Dónde ocurrió (la cadena de llamadas) |
| `InnerException` | La excepción original, si esta envuelve a otra |
| `GetType().Name` | El tipo concreto (`DivideByZeroException`) |
| `ToString()` | Todo lo anterior junto (lo ideal para un log) |

### Varios `catch`: de lo específico a lo general

```csharp
try
{
    Console.Write("Número: ");
    int numero = int.Parse(Console.ReadLine()!);
    Console.WriteLine($"100 / {numero} = {100 / numero}");
}
catch (FormatException)
{
    Console.WriteLine("Eso no es un número válido.");
}
catch (DivideByZeroException)
{
    Console.WriteLine("No se puede dividir por cero.");
}
catch (Exception ex)
{
    Console.WriteLine($"Error inesperado: {ex.Message}");
}
```

* Se evalúan **en orden** y se ejecuta **el primero** compatible.
* `Exception` es la base de todas: atrapa cualquier cosa, así que va **al final**. Si lo pones primero, los siguientes nunca se alcanzan y el compilador da error (CS0160).
* Si no usas la variable, puedes omitirla: `catch (FormatException)`.

### `finally`: se ejecuta siempre

```csharp
StreamReader? lector = null;
try
{
    lector = new StreamReader("datos.txt");
    Console.WriteLine(lector.ReadLine());
}
catch (FileNotFoundException)
{
    Console.WriteLine("No se encontró el archivo.");
}
finally
{
    lector?.Dispose();                 // se ejecuta haya o no excepción
    Console.WriteLine("Recursos liberados");
}
```

`finally` corre **siempre**: si el `try` terminó bien, si hubo una excepción capturada, si hubo una no capturada (antes de propagarse) e incluso si dentro del `try` hay un `return`. Es el lugar para liberar recursos.

`finally` es opcional, y `try` + `finally` sin `catch` también es válido: limpias, pero dejas que la excepción siga su camino.

### `using`: el `finally` automático

Para objetos `IDisposable` (archivos, conexiones), `using` genera el `try`/`finally` por ti:

```csharp
using (var lector = new StreamReader("datos.txt"))
{
    Console.WriteLine(lector.ReadLine());
}   // aquí se llama a lector.Dispose(), aunque haya habido una excepción

using var otro = new StreamReader("otro.txt");   // forma corta (C# 8): se libera al final del bloque actual
```

Prefiere `using` a un `finally` escrito a mano.

### Excepciones habituales

| Excepción | Cuándo ocurre | Ejemplo |
| --- | --- | --- |
| `NullReferenceException` | Usar un miembro de una referencia `null` | `string? s = null; s.Length` |
| `IndexOutOfRangeException` | Índice inválido en un array | `arr[arr.Length]` |
| `ArgumentOutOfRangeException` | Índice o argumento fuera de rango (listas, strings, métodos) | `lista[99]`, `"abc".Substring(5)` |
| `ArgumentException` / `ArgumentNullException` | Argumento inválido o `null` | `dic.Add(claveRepetida, v)` |
| `FormatException` | Texto con formato incorrecto | `int.Parse("hola")` |
| `DivideByZeroException` | División entera por cero | `10 / 0` (con variables) |
| `OverflowException` | Desbordamiento en contexto `checked` | `checked(int.MaxValue + x)` |
| `InvalidOperationException` | Operación inválida en el estado actual | `First()` sobre vacía, modificar una lista en un `foreach` |
| `KeyNotFoundException` | Clave inexistente en un diccionario | `dic["nada"]` |
| `FileNotFoundException`, `IOException` | Problemas con archivos | abrir un archivo que no existe |
| `InvalidCastException` | Cast imposible | `(string)(object)42` |

Ojo con un mito común: `List<T>.Remove(item)` **no** lanza excepción si el elemento no existe; devuelve `false`.

### Propagación por la pila de llamadas

Si un método no captura una excepción, esta **sube** al método que lo llamó, y así sucesivamente:

```csharp
try
{
    Procesar();                     // 3. aquí se captura
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Capturada en el nivel superior: {ex.Message}");
}

static void Procesar() => Validar(-5);                     // 2. no la captura: sigue subiendo

static void Validar(int cantidad)
{
    if (cantidad < 0)
        throw new InvalidOperationException("Cantidad negativa");   // 1. se lanza aquí
}
```

```text
Main  ──llama──►  Procesar  ──llama──►  Validar
  ▲                                        │
  └──────────── la excepción sube ─────────┘  (Procesar no la maneja)
```

Esto es **deseable**: el método que detecta el problema (`Validar`) no siempre sabe qué hacer con él. Quien está más arriba tiene el contexto para decidir: en una consola, volver a preguntar; en una web, devolver un error 400; en un servicio, registrar y reintentar.

Si la excepción llega hasta el punto de entrada sin que nadie la capture, el proceso termina con "Unhandled exception" y la traza de pila.

### Filtros de excepción: `when`

Un `catch` puede tener una condición: solo maneja la excepción si se cumple. Si no, la excepción sigue de largo hacia el siguiente `catch` o hacia arriba:

```csharp
try
{
    var respuesta = await cliente.GetStringAsync(url);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    Console.WriteLine("El recurso no existe. Revisa la URL.");
}
catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)
{
    Console.WriteLine("El servidor falló. Intenta más tarde.");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"Error de red: {ex.Message}");
}
```

(`await` se ve en el módulo de asincronía; aquí lo importante es el `when`). Un filtro es mejor que capturar y volver a lanzar dentro del `catch`, porque si la condición es falsa la excepción ni siquiera se "toca" y conserva intacta su traza.

### No te tragues las excepciones

```csharp
try
{
    GuardarPedido(pedido);
}
catch (Exception)
{
    // vacío: "que no se caiga"
}
```

Este es el peor manejo posible: el pedido no se guardó, nadie se entera, y el bug aparece semanas después en otro lugar, sin pistas. Si capturas una excepción, haz algo con ella: muéstrala, regístrala, recupérate o vuelve a lanzarla.

-----

## Ejemplo completo

Una calculadora de consola que nunca se cae y siempre informa qué pasó:

```csharp
while (true)
{
    Console.Write("Expresión (ej. 10 / 2) o 'salir': ");
    string? entrada = Console.ReadLine();
    if (entrada is null || entrada.Trim().Equals("salir", StringComparison.OrdinalIgnoreCase)) break;

    try
    {
        int resultado = Calcular(entrada);
        Console.WriteLine($"= {resultado}");
    }
    catch (FormatException ex)
    {
        Console.WriteLine($"Formato inválido: {ex.Message}");
    }
    catch (DivideByZeroException)
    {
        Console.WriteLine("No se puede dividir por cero.");
    }
    catch (OverflowException)
    {
        Console.WriteLine("El resultado es demasiado grande.");
    }
    catch (NotSupportedException ex) when (ex.Message.Contains('%'))
    {
        Console.WriteLine("El módulo todavía no está soportado.");
    }
    finally
    {
        Console.WriteLine("---");
    }
}

Console.WriteLine("¡Hasta luego!");

static int Calcular(string expresion)
{
    string[] partes = expresion.Split(' ', StringSplitOptions.RemoveEmptyEntries);
    if (partes.Length != 3)
        throw new FormatException("Usa el formato: número operador número.");

    int a = int.Parse(partes[0]);     // puede lanzar FormatException u OverflowException
    int b = int.Parse(partes[2]);

    return partes[1] switch
    {
        "+" => checked(a + b),
        "-" => checked(a - b),
        "*" => checked(a * b),
        "/" => a / b,                 // puede lanzar DivideByZeroException
        _ => throw new NotSupportedException($"Operador '{partes[1]}' no soportado.")
    };
}
```

Ejecución de ejemplo:

```text
Expresión (ej. 10 / 2) o 'salir': 10 / 2
= 5
---
Expresión (ej. 10 / 2) o 'salir': 7 / 0
No se puede dividir por cero.
---
Expresión (ej. 10 / 2) o 'salir': 2000000000 * 2
El resultado es demasiado grande.
---
Expresión (ej. 10 / 2) o 'salir': hola
Formato inválido: Usa el formato: número operador número.
---
Expresión (ej. 10 / 2) o 'salir': salir
¡Hasta luego!
```

`Calcular` no captura nada: detecta y lanza. El bucle principal, que sabe hablar con el usuario, decide qué mostrar. (Si escribes `5 % 2`, la excepción `NotSupportedException` tiene el operador en el mensaje y la captura el filtro; con otro operador desconocido, como `5 ^ 2`, no la captura ningún `catch` y el programa termina: prueba qué pasa).

-----

## Errores comunes

**1. `catch (Exception)` antes de los específicos.**
Qué pasa: `error CS0160: A previous catch clause already catches all exceptions of this or of a super type ('Exception')`.
Por qué: el `catch` general atrapa todo y los siguientes serían inalcanzables.
Arreglo: ordena de lo más específico a lo más general.

**2. Tragarse la excepción.**
Qué pasa: el programa sigue "como si nada" con datos incorrectos o incompletos.
Por qué: un `catch` vacío oculta el error.
Arreglo: registra, informa, recupérate o vuelve a lanzar con `throw;`.

**3. Envolver todo el programa en un único `try/catch (Exception)`.**
Qué pasa: un mensaje genérico ("ocurrió un error") para cualquier problema.
Por qué: no distingue qué falló ni cómo recuperarse.
Arreglo: captura excepciones concretas donde puedes hacer algo útil; deja un manejador global solo para registrar lo inesperado.

**4. Usar excepciones para casos esperados.**
Qué pasa: código lento y confuso: `try { int.Parse(x) } catch { ... }` en cada entrada del usuario.
Por qué: lanzar y capturar excepciones es costoso, y un dato inválido del usuario no es "excepcional".
Arreglo: `int.TryParse`, `dic.TryGetValue`, `File.Exists`, `FirstOrDefault`.

**5. Usar una variable declarada dentro del `try` en el `finally`.**
Qué pasa: `error CS0103: The name 'lector' does not exist in the current context`.
Por qué: el `try` es un bloque con su propio ámbito.
Arreglo: declárala antes del `try` (o, mejor, usa `using`).

**6. Mostrar el `StackTrace` al usuario final.**
Qué pasa: el usuario ve detalles internos (rutas, nombres de clases) que además pueden ser un riesgo de seguridad.
Por qué: la traza es para desarrolladores.
Arreglo: mensaje amigable para el usuario y `ex.ToString()` completo en el log.

-----

## Según la versión de C#

* **C# 1:** `try`, `catch`, `finally` y `throw`.
* **C# 6:** filtros de excepción (`when`) y `await` dentro de `catch` y `finally`.
* **C# 7:** expresiones `throw` (`?? throw new ...`).
* **C# 8:** declaraciones `using` sin llaves (`using var x = ...;`).
* **.NET Core 3+:** las excepciones no controladas en .NET muestran trazas más legibles, incluidos los métodos `async`.

-----

## Cuándo sí y cuándo no

**Captura una excepción cuando:**

* Puedes **hacer algo útil**: reintentar, usar un valor alternativo, informar al usuario, convertirla en una respuesta de error.
* Estás en una **frontera** (la interfaz con el usuario, un endpoint de API, un worker) y necesitas registrar el error y responder ordenadamente.

**No la captures cuando:**

* No sabes qué hacer con ella: déjala propagar a quien tenga contexto.
* El caso es esperado y existe una alternativa sin excepciones (`TryParse`, `TryGetValue`).

**Usa `finally` o `using` siempre que:**

* Abras recursos que deben cerrarse: archivos, conexiones, bloqueos.

-----

## Resumen en 5 líneas

1. `try` contiene el código que puede fallar; `catch (TipoDeExcepcion ex)` lo maneja; `finally` se ejecuta siempre.
2. Los `catch` se evalúan en orden: de lo específico a lo general (`Exception` al final).
3. Una excepción no capturada sube por la pila de llamadas; si llega al punto de entrada, termina el programa.
4. `catch (...) when (condición)` filtra sin perder la traza; `using` reemplaza el `finally` para objetos `IDisposable`.
5. Nunca te tragues una excepción, y no las uses para casos esperados: para eso existen los métodos `Try...`.

-----

## Para profundizar

<details>
<summary>Manejo global de excepciones</summary>

En una aplicación de consola puedes registrar las excepciones que nadie capturó:

```csharp
AppDomain.CurrentDomain.UnhandledException += (_, e) =>
    Console.Error.WriteLine($"Error fatal: {e.ExceptionObject}");
```

No evita que el proceso termine, pero permite dejar un registro. En ASP.NET Core se usa un middleware de manejo de excepciones (`app.UseExceptionHandler(...)`) que convierte cualquier excepción no controlada en una respuesta HTTP 500 con formato `ProblemDetails`, y la registra en el log.

</details>

<details>
<summary>Cuánto cuesta una excepción</summary>

Lanzar una excepción implica crear el objeto, capturar la traza de pila y recorrer la pila buscando un `catch`. Es miles de veces más lento que un `if`. No importa en errores reales (que ocurren pocas veces), pero sí si se usan como control de flujo en un bucle con miles de iteraciones. Por eso .NET ofrece el patrón `TryX`. Un bloque `try` sin excepciones, en cambio, prácticamente no tiene costo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una excepción es un error en tiempo de ejecución. Se maneja con `try`/`catch`: el código que puede fallar va en el `try`, y en el `catch` se captura el tipo de excepción y se reacciona. El bloque `finally` se ejecuta siempre y se usa para liberar recursos. Si una excepción no se captura, sube por la pila de llamadas hasta detener el programa.

### Respuesta ampliada (semi-senior)

Las excepciones derivan de `System.Exception` y se propagan por la pila hasta encontrar un `catch` compatible; los bloques se evalúan en orden, por lo que van de lo específico a lo general. `finally` garantiza la limpieza (y `using` lo genera para `IDisposable`). Los filtros `when` se evalúan antes de desenrollar la pila, así que no alteran la traza y permiten registrar sin capturar. La estrategia es capturar solo donde se puede actuar o en las fronteras de la aplicación (manejador global, middleware), no tragarse errores y no usar excepciones para el control de flujo de casos esperados, para los que existen los patrones `TryX` o tipos `Result`.

### Preguntas frecuentes de seguimiento

**1. ¿Se ejecuta el `finally` si hay un `return` en el `try`?**
Sí. El `finally` se ejecuta antes de que el método termine de devolver el valor.

**2. ¿Por qué el `catch (Exception)` va al final?**
Porque atrapa cualquier excepción; si estuviera antes, los `catch` más específicos nunca se alcanzarían (el compilador da CS0160).

**3. ¿Qué ventaja tiene un filtro `when`?**
Permite capturar solo bajo cierta condición sin desenrollar la pila ni perder la traza si la condición no se cumple.

-----

## Práctica

**Ejercicio 1.** Escribe un programa que pida un índice al usuario y muestre el elemento correspondiente de `string[] frutas = { "manzana", "pera", "uva" };`. Maneja por separado: entrada que no es un número y un índice fuera de rango. Usa `finally` para mostrar "Consulta terminada".

<details>
<summary>Solución</summary>

```csharp
string[] frutas = { "manzana", "pera", "uva" };

try
{
    Console.Write("Índice: ");
    int i = int.Parse(Console.ReadLine() ?? "");
    Console.WriteLine(frutas[i]);
}
catch (FormatException)
{
    Console.WriteLine("Debes escribir un número.");
}
catch (IndexOutOfRangeException)
{
    Console.WriteLine($"El índice debe estar entre 0 y {frutas.Length - 1}.");
}
finally
{
    Console.WriteLine("Consulta terminada");
}
```

Mejor todavía: valida con `int.TryParse` y `i >= 0 && i < frutas.Length` y deja las excepciones para lo realmente inesperado. Practica las dos versiones.

</details>

**Ejercicio 2.** ¿Qué imprime este programa?

```csharp
try
{
    Console.WriteLine("A");
    Metodo();
    Console.WriteLine("B");
}
catch (InvalidOperationException)
{
    Console.WriteLine("C");
}
finally
{
    Console.WriteLine("D");
}
Console.WriteLine("E");

static void Metodo()
{
    try
    {
        throw new InvalidOperationException();
    }
    finally
    {
        Console.WriteLine("F");
    }
}
```

<details>
<summary>Solución</summary>

```text
A
F
C
D
E
```

`Metodo` lanza; su `finally` imprime F antes de que la excepción suba. "B" se salta. El `catch` de afuera imprime C, el `finally` imprime D y el programa continúa con E.

</details>

-----

## Siguiente lección

[Lanzar y crear excepciones](02-Lanzar%20y%20crear%20excepciones.md)
