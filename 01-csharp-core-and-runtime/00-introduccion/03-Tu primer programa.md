# Tu primer programa

## En una frase

Un programa de consola en C# escribe texto con `Console.WriteLine()`, lo lee con `Console.ReadLine()` y puede llevar comentarios (`//`, `/* */`) que el compilador ignora.

-----

## Antes de empezar

Conviene que ya sepas:

* Crear y ejecutar un proyecto con `dotnet new console` y `dotnet run`, como en [Preparar el entorno](02-Preparar%20el%20entorno.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Consola (terminal):** la ventana de texto donde el programa muestra resultados y recibe lo que escribe el usuario.
* **Sentencia:** una instrucción completa. En C# termina en punto y coma `;`.
* **Método:** un bloque de código con nombre que realiza una tarea. `WriteLine` es un método.
* **Argumento:** el valor que le pasas a un método entre paréntesis.
* **String (cadena):** un texto, escrito entre comillas dobles: `"Hola"`.
* **Comentario:** texto dentro del código que el compilador ignora.
* **Punto de entrada:** el lugar donde empieza a ejecutarse el programa.
* **Top-level statements:** sentencias escritas directamente en `Program.cs`, sin declarar una clase ni un método `Main`.

-----

## El problema

Un programa que no muestra nada ni recibe nada no sirve para aprender, porque no ves qué está pasando. Necesitas dos canales básicos: **salida** (mostrar resultados) y **entrada** (recibir datos del usuario).

Además, el código lo leen personas, no solo la computadora. Necesitas una forma de dejar explicaciones que no afecten a la ejecución.

-----

## Cómo funciona

### El punto de entrada

En un proyecto moderno, `Program.cs` puede contener solo esto:

```csharp
Console.WriteLine("¡Hola, mundo!");
```

Eso es un programa completo. Se llama **top-level statements**: el compilador genera por ti la clase y el método de entrada. Lo que escribiste es equivalente a esta versión "larga", que verás en tutoriales y proyectos antiguos:

```csharp
using System;

namespace MiApp
{
    internal class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("¡Hola, mundo!");
        }
    }
}
```

* `using System;` permite escribir `Console` en lugar de `System.Console`.
* `namespace` agrupa las clases bajo un nombre.
* `class Program` es una clase; las clases se estudian en [Programación orientada a objetos](../04-poo/README.md).
* `static void Main(string[] args)` es el **punto de entrada**: el primer método que se ejecuta. Se explica parte por parte en [Miembros estáticos](../04-poo/05-Miembros%20estaticos.md).

Las dos formas producen lo mismo. Un proyecto solo puede tener **un** archivo con top-level statements.

### Mostrar texto: `WriteLine` y `Write`

```csharp
Console.WriteLine("Primera línea");   // escribe y salta de línea
Console.Write("Hola");                // escribe sin saltar de línea
Console.Write(" Mundo");
Console.WriteLine();                  // solo salta de línea
```

Salida:

```text
Primera línea
Hola Mundo
```

`Console` es una clase y `WriteLine` es un método de esa clase. El punto (`.`) significa "el miembro `WriteLine` que pertenece a `Console`".

### Caracteres especiales en el texto

La barra invertida `\` inicia una **secuencia de escape**: le dice al compilador que el siguiente carácter tiene un significado especial.

```csharp
Console.WriteLine("Ella dijo: \"Hola\"");   // Ella dijo: "Hola"
Console.WriteLine("Línea 1\nLínea 2");       // salto de línea
Console.WriteLine("Columna\tColumna");       // tabulación
Console.WriteLine("C:\\Usuarios\\Ana");      // C:\Usuarios\Ana
```

La tabla completa de secuencias está en [Texto: char y string](../01-tipos-y-variables/05-Texto%20char%20y%20string.md).

### Leer lo que escribe el usuario: `ReadLine`

```csharp
Console.WriteLine("¿Cómo te llamas?");
string nombre = Console.ReadLine();
Console.WriteLine($"Hola, {nombre}.");
```

1. El programa muestra la pregunta.
2. `Console.ReadLine()` **detiene** el programa hasta que el usuario escribe algo y presiona Enter.
3. El texto escrito se guarda en la variable `nombre`, de tipo `string`. Las variables se estudian en [Variables y tipos de datos](../01-tipos-y-variables/01-Variables%20y%20tipos%20de%20datos.md).
4. `$"Hola, {nombre}."` es una **cadena interpolada**: el `$` permite insertar valores entre llaves.

`ReadLine()` **siempre devuelve texto**. Si el usuario escribe `25`, recibes el texto `"25"`, no el número 25. Convertirlo se explica en [Conversiones de tipos](../01-tipos-y-variables/03-Conversiones%20de%20tipos.md).

### La advertencia de nulos y el operador `!`

Con `<Nullable>enable</Nullable>` (el valor por defecto), la línea `string nombre = Console.ReadLine();` produce una **advertencia**:

```text
warning CS8600: Converting null literal or possible null value to non-nullable type.
```

`ReadLine()` puede devolver `null` (por ejemplo, si la entrada se cierra con Ctrl + Z o Ctrl + D, o si se redirige desde un archivo vacío). Tienes tres opciones:

```csharp
string? nombre1 = Console.ReadLine();          // 1. aceptar que puede ser null (string?)
string nombre2 = Console.ReadLine() ?? "";     // 2. dar un valor por defecto si es null
string nombre3 = Console.ReadLine()!;          // 3. "confía en mí, no es null"
```

El `!` se llama **operador que perdona nulos** (*null-forgiving operator*). **No cambia nada en ejecución**: solo silencia la advertencia. Si el valor es `null`, el programa fallará igual más adelante. Prefiere la opción 2 cuando no estés seguro. El tema completo está en [Tipos que aceptan null](../05-tipos-avanzados/03-Tipos%20que%20aceptan%20null.md).

### Comentarios

```csharp
// Comentario de una línea

/* Comentario
   de varias líneas */

/// <summary>
/// Comentario de documentación XML: describe un método o una clase.
/// El editor lo muestra al pasar el mouse sobre su nombre.
/// </summary>
```

Los comentarios sirven para:

* Explicar **por qué** el código hace algo de cierta forma (el **qué** ya lo dice el código).
* Desactivar una línea temporalmente para probar sin ella.
* Documentar la API pública de una librería (con `///`).

```csharp
// Usamos 365 días: el cliente no considera años bisiestos en el cálculo de intereses.
const int DiasPorAnio = 365;
```

Ese comentario aporta algo que el código no puede decir. Este, en cambio, sobra:

```csharp
// suma 1 a contador
contador++;
```

-----

## Ejemplo completo

```csharp
// Programa que saluda y calcula el año de nacimiento aproximado.

Console.WriteLine("=== Bienvenido ===");

Console.Write("¿Cómo te llamas? ");
string nombre = Console.ReadLine() ?? "desconocido";

Console.Write("¿Cuántos años tienes? ");
string textoEdad = Console.ReadLine() ?? "0";
int edad = int.Parse(textoEdad);   // convierte el texto en número (se ve en Conversiones de tipos)

int anioActual = DateTime.Now.Year;
Console.WriteLine($"Hola, {nombre}. Naciste alrededor de {anioActual - edad}.");
```

Ejecución de ejemplo:

```text
=== Bienvenido ===
¿Cómo te llamas? Ana
¿Cuántos años tienes? 30
Hola, Ana. Naciste alrededor de 1996.
```

-----

## Errores comunes

**1. Olvidar el punto y coma.**
Qué pasa: `error CS1002: ; expected`.
Por qué: en C#, cada sentencia termina en `;`.
Arreglo: agrega `;` al final de la línea que indica el error.

**2. Escribir `console.writeline` en minúsculas.**
Qué pasa: `error CS0103: The name 'console' does not exist in the current context`.
Por qué: C# distingue mayúsculas de minúsculas. `Console` y `console` son nombres distintos.
Arreglo: `Console.WriteLine`.

**3. Usar comillas simples para un texto.**
Qué pasa: `error CS1012: Too many characters in character literal`.
Por qué: `'a'` es un solo carácter (`char`); los textos van entre comillas dobles.
Arreglo: `"Hola"`.

**4. Olvidar escapar comillas internas.**
Qué pasa: errores de compilación en cascada, porque el texto se "cierra" antes de tiempo.
Por qué: la segunda `"` termina el string.
Arreglo: `\"` dentro del texto.

**5. Tratar el resultado de `ReadLine()` como número.**
Qué pasa: `error CS0029: Cannot implicitly convert type 'string' to 'int'`.
Por qué: `ReadLine()` devuelve `string`.
Arreglo: convertir con `int.Parse` o, mejor, `int.TryParse`. Se ve en [Conversiones de tipos](../01-tipos-y-variables/03-Conversiones%20de%20tipos.md).

**6. Tener top-level statements en dos archivos.**
Qué pasa: `error CS8802: Only one compilation unit can have top-level statements`.
Por qué: solo puede haber un punto de entrada.
Arreglo: deja las sentencias sueltas en un solo archivo (normalmente `Program.cs`).

-----

## Según la versión de C#

* **C# 6:** interpolación de cadenas con `$"...{valor}..."`. Antes se usaba `string.Format("Hola, {0}", nombre)`.
* **C# 8:** tipos de referencia que aceptan nulos (`string?`) y el operador `!`.
* **C# 9:** top-level statements. Antes, todo programa necesitaba `class Program` y `static void Main`.
* **C# 10:** `global using` e `ImplicitUsings`: ya no escribes `using System;` en cada archivo.

-----

## Cuándo sí y cuándo no

**Usa top-level statements cuando:**

* Escribes programas de consola, ejemplos o el `Program.cs` de una API (así lo genera ASP.NET Core).

**Usa `Main` explícito cuando:**

* El equipo o el proyecto ya sigue ese estilo.
* Quieres mostrar de forma explícita la estructura de clases (por ejemplo, en material de enseñanza de POO).

**Comentarios:**

* Úsalos para explicar decisiones, restricciones o el porqué.
* Evita comentar lo obvio o dejar código comentado "por si acaso": para eso está el control de versiones (git).

-----

## Resumen en 5 líneas

1. `Console.WriteLine` escribe y salta de línea; `Console.Write` escribe sin saltar.
2. `Console.ReadLine()` espera al usuario y devuelve un `string` (o `null`).
3. `\"`, `\n`, `\t` y `\\` son secuencias de escape dentro de un texto.
4. Top-level statements y `static void Main` son dos formas del mismo punto de entrada.
5. Comentarios: `//` una línea, `/* */` varias, `///` documentación XML.

-----

## Para profundizar

<details>
<summary>Qué genera el compilador con top-level statements</summary>

El compilador crea una clase llamada `Program` con un método de entrada cuyo nombre interno es `<Main>$`. Si usas `await` en las sentencias, el método generado devuelve `Task`; si usas `return 1;`, devuelve `int` (código de salida del proceso). Las variables que declaras ahí son **locales** de ese método, no campos de clase. Por eso un método declarado en `Program.cs` con top-level statements es en realidad una **función local**.

</details>

<details>
<summary>Entrada y salida estándar</summary>

`Console.WriteLine` escribe en la **salida estándar** (stdout) y `Console.ReadLine` lee de la **entrada estándar** (stdin). Para los errores existe `Console.Error.WriteLine` (stderr). Esto permite redirigir desde la terminal: `dotnet run < entrada.txt > salida.txt`. Cuando stdin llega al final, `ReadLine` devuelve `null`, y por eso el tipo de retorno es `string?`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`Console.WriteLine` imprime texto en la consola y `Console.ReadLine` lee una línea que escribe el usuario, siempre como `string`. Desde C# 9 se pueden usar top-level statements, que evitan escribir la clase `Program` y el método `Main`.

### Respuesta ampliada (semi-senior)

El punto de entrada es `Main`, que puede ser `void` o `int` y opcionalmente `async Task`/`Task<int>`, con o sin `string[] args`. Con top-level statements, el compilador sintetiza esa clase y ese método; solo puede haber un archivo así por proyecto. `ReadLine` devuelve `string?` porque puede llegar al fin de la entrada; con nullable activado conviene manejarlo con `??` o una comprobación, y no silenciarlo con `!`, que no cambia el comportamiento en ejecución.

### Preguntas frecuentes de seguimiento

**1. ¿Qué hace el operador `!` en `Console.ReadLine()!`?**
Le dice al compilador que no muestre la advertencia de nulos. No convierte nada ni evita un `NullReferenceException` posterior.

**2. ¿Puede `Main` devolver un valor?**
Sí: `static int Main()` devuelve un código de salida al sistema operativo (0 suele significar éxito).

**3. ¿Para qué sirven los comentarios `///`?**
Generan documentación XML que el IDE muestra en IntelliSense y que herramientas como DocFX convierten en documentación web.

-----

## Práctica

**Ejercicio 1.** Escribe un programa que pida al usuario su ciudad y su comida favorita, y luego imprima en **una sola línea**: `Vives en <ciudad> y te gusta <comida>.`

<details>
<summary>Solución</summary>

```csharp
Console.Write("¿En qué ciudad vives? ");
string ciudad = Console.ReadLine() ?? "";

Console.Write("¿Cuál es tu comida favorita? ");
string comida = Console.ReadLine() ?? "";

Console.WriteLine($"Vives en {ciudad} y te gusta {comida}.");
```

</details>

**Ejercicio 2.** Sin ejecutarlo, ¿qué imprime este código?

```csharp
Console.Write("A");
Console.WriteLine("B");
Console.Write("C\tD\n");
Console.WriteLine("\"E\"");
```

<details>
<summary>Solución</summary>

```text
AB
C	D
"E"
```

`Write("A")` no salta de línea, por eso `A` y `B` quedan juntos. `\t` es un tabulador y `\n` un salto de línea.

</details>

-----

## Siguiente lección

[Paradigmas de programación](04-Paradigmas%20de%20programacion.md)
