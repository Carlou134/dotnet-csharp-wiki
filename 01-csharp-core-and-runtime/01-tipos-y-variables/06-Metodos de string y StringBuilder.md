# Métodos de string y StringBuilder

## En una frase

`string` trae métodos para **consultar** (`Length`, `IndexOf`, `Contains`), **extraer** (`Substring`, `[índice]`, rangos) y **transformar** (`ToUpper`, `Trim`, `Replace`, `Split`), que siempre devuelven un resultado nuevo; cuando necesitas construir texto en muchos pasos, `StringBuilder` lo hace sin crear un string por cada paso.

-----

## Antes de empezar

Conviene que ya sepas:

* Que los strings son inmutables y se indexan desde 0, de [Texto: char y string](05-Texto%20char%20y%20string.md).
* Qué es un método y cómo se llama con un punto: `texto.ToUpper()`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Índice:** la posición de un carácter dentro del string, empezando en 0.
* **Subcadena (*substring*):** una parte de un string.
* **Propiedad:** un dato que expone un objeto, sin paréntesis: `texto.Length`.
* **Sobrecarga:** varias versiones de un mismo método con distintos parámetros (por ejemplo, `IndexOf(char)` e `IndexOf(string)`).
* **`StringBuilder`:** clase con un búfer modificable para construir texto de forma eficiente.
* **Comparación ordinal:** comparar carácter por carácter según su código, sin reglas de idioma.

-----

## El problema

Imagina que procesas los datos de un formulario:

* El usuario escribió `"   ana.torres@MAIL.com  "`, con espacios y mayúsculas mezcladas.
* Necesitas el dominio del correo (lo que va después de la `@`).
* Tienes que validar que el tweet no supere los 280 caracteres.
* Al final, generar un reporte de 10.000 líneas.

Hacer todo eso recorriendo caracteres a mano sería largo y propenso a errores. .NET ya trae los métodos; solo hay que conocerlos y entender que **ninguno modifica el original**.

-----

## Cómo funciona

### Consultar: longitud y búsqueda

```csharp
string texto = "hola mundo";

int largo = texto.Length;                   // 10 (propiedad: sin paréntesis)

texto.IndexOf('o');                         // 1  primera aparición
texto.IndexOf("mundo");                     // 5
texto.IndexOf('z');                         // -1 no existe
texto.IndexOf('o', 2);                      // 9  busca desde la posición 2
texto.LastIndexOf('o');                     // 9  última aparición
"abc123".IndexOfAny(new[] { '1', '2' });    // 3  el primero que encuentre de varios

texto.Contains("mun");                      // True
texto.StartsWith("hola");                   // True
texto.EndsWith("!");                        // False
```

Reglas de `IndexOf`:

* Devuelve la posición de la **primera** coincidencia.
* Devuelve **-1** si no la encuentra. Compruébalo siempre antes de usar el resultado.
* `IndexOf("")` devuelve 0.

### Extraer partes

```csharp
string planta = "Cactaceae, Cactus";

int inicio = planta.IndexOf("Cactus");        // 11
string comun = planta.Substring(inicio);      // "Cactus" (desde 11 hasta el final)

string nombre = "Codecademy";
string parte = nombre.Substring(2, 6);        // "decade" (desde 2, 6 caracteres)

char letra = nombre[0];                       // 'C'
char ultima = nombre[^1];                     // 'y'
string rango = nombre[2..8];                  // "decade" (desde 2 hasta 8, sin incluir 8)
string primeras = nombre[..4];                // "Code"
string desde = nombre[4..];                   // "cademy"
```

`Substring(inicio, longitud)` recibe una **longitud**; el rango `[inicio..fin]` recibe una **posición final exclusiva**. Es fácil confundirlos.

### Transformar

Todos devuelven un string **nuevo**:

```csharp
"Hola".ToUpper();                  // "HOLA"
"Hola".ToLower();                  // "hola"
"  Hola  ".Trim();                 // "Hola"   quita espacios al inicio y al final
"  Hola  ".TrimStart();            // "Hola  "
"--Hola--".Trim('-');              // "Hola"   quita los caracteres indicados
"42".PadLeft(5);                   // "   42"  rellena a la izquierda hasta 5 caracteres
"42".PadLeft(6, '0');              // "000042"
"Hola".PadRight(6, '.');           // "Hola.."
"Hola Mundo".Replace("Mundo", "C#"); // "Hola C#"
"HolaMundo".Remove(4);             // "Hola"   elimina desde la posición 4
"HolaMundo".Remove(4, 2);          // "Holando" elimina 2 caracteres desde la 4
"Hola".Insert(4, " C#");           // "Hola C#"
```

### Dividir y unir

```csharp
string csv = "manzana,pera,plátano";
string[] frutas = csv.Split(',');                 // ["manzana", "pera", "plátano"]

string frase = "  uno  dos   tres ";
string[] palabras = frase.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
// ["uno", "dos", "tres"]

string unido = string.Join(" - ", frutas);       // "manzana - pera - plátano"
char[] letras = "Hola".ToCharArray();            // ['H', 'o', 'l', 'a']
```

Sin `RemoveEmptyEntries`, los espacios repetidos generan entradas vacías (`""`) en el array.

### Validaciones frecuentes

```csharp
string.IsNullOrEmpty(texto);        // true si es null o ""
string.IsNullOrWhiteSpace(texto);   // true si es null, "" o solo espacios → la más útil para formularios
```

### Comparaciones y mayúsculas

Para comparar sin distinguir mayúsculas, **no** hagas `a.ToLower() == b.ToLower()`: crea dos strings nuevos y puede fallar con algunos idiomas. Usa `StringComparison`:

```csharp
string rol = "ADMIN";

rol.Equals("admin", StringComparison.OrdinalIgnoreCase);       // True
rol.Contains("adm", StringComparison.OrdinalIgnoreCase);       // True
rol.StartsWith("ad", StringComparison.OrdinalIgnoreCase);      // True
```

| Opción | Úsala para |
| --- | --- |
| `Ordinal` | Identificadores, claves, rutas, protocolos: comparación exacta y rápida. |
| `OrdinalIgnoreCase` | Lo mismo, sin distinguir mayúsculas (correos, nombres de usuario, roles). |
| `CurrentCulture` | Texto que ve una persona y debe ordenarse según su idioma. |

### StringBuilder: construir texto en muchos pasos

Como los strings son inmutables, este bucle crea **10.000 strings intermedios**, cada uno más largo que el anterior:

```csharp
string reporte = "";
for (int i = 0; i < 10_000; i++)
{
    reporte += $"Línea {i}\n";      // cada += copia todo el texto anterior en un string nuevo
}
```

`StringBuilder` mantiene un **búfer modificable** y solo crea el string final al llamar a `ToString()`:

```csharp
using System.Text;

var sb = new StringBuilder();
for (int i = 0; i < 10_000; i++)
{
    sb.AppendLine($"Línea {i}");
}
string reporte = sb.ToString();
```

Métodos principales:

```csharp
var sb = new StringBuilder("Hola");
sb.Append(" mundo");        // "Hola mundo"
sb.AppendLine("!");         // "Hola mundo!\n" (agrega un salto de línea)
sb.Insert(0, ">> ");        // ">> Hola mundo!\n"
sb.Replace("mundo", "C#");  // ">> Hola C#!\n"
sb.Remove(0, 3);            // "Hola C#!\n"
int largo = sb.Length;
string final = sb.ToString();
```

Los métodos de `StringBuilder` **sí modifican** el objeto (y además lo devuelven, por eso se pueden encadenar: `sb.Append("a").Append("b");`).

-----

## Ejemplo completo

```csharp
using System.Text;

string entrada = "   ana.torres@MAIL.com  ";
string tweet = "Aprendiendo C# con métodos de string #dotnet";

// 1. Normalizar el correo
string correo = entrada.Trim().ToLowerInvariant();

// 2. Extraer usuario y dominio
int arroba = correo.IndexOf('@');
if (arroba == -1)
{
    Console.WriteLine("Correo inválido");
    return;
}
string usuario = correo[..arroba];
string dominio = correo[(arroba + 1)..];

// 3. Validar el tweet
const int MaxCaracteres = 280;
bool tweetValido = tweet.Length <= MaxCaracteres;
string[] hashtags = tweet.Split(' ').Where(p => p.StartsWith('#')).ToArray();

// 4. Construir el reporte
var sb = new StringBuilder();
sb.AppendLine("=== Reporte ===");
sb.AppendLine($"Correo : {correo}");
sb.AppendLine($"Usuario: {usuario.Replace('.', ' ')}");
sb.AppendLine($"Dominio: {dominio.ToUpper()}");
sb.AppendLine($"Tweet  : {tweet.Length}/{MaxCaracteres} {(tweetValido ? "OK" : "DEMASIADO LARGO")}");
sb.AppendLine($"Tags   : {string.Join(", ", hashtags)}");

Console.Write(sb.ToString());
```

Salida:

```text
=== Reporte ===
Correo : ana.torres@mail.com
Usuario: ana torres
Dominio: MAIL.COM
Tweet  : 44/280 OK
Tags   : #dotnet
```

`Where` y `ToArray` son de LINQ; se ven más adelante. Aquí solo filtran las palabras que empiezan con `#`.

-----

## Errores comunes

**1. No guardar el resultado.**
Qué pasa: `nombre.Trim();` no tiene efecto.
Por qué: los métodos de `string` devuelven un string nuevo.
Arreglo: `nombre = nombre.Trim();`.

**2. Usar el resultado de `IndexOf` sin comprobar -1.**
Qué pasa: `System.ArgumentOutOfRangeException` en `Substring(-1)`.
Por qué: el texto buscado no existía.
Arreglo: `if (pos >= 0) { ... }`.

**3. Pasarse del final con `Substring`.**
Qué pasa: `ArgumentOutOfRangeException: Index and length must refer to a location within the string`.
Por qué: `inicio + longitud` supera `Length`.
Arreglo: verifica la longitud o usa `Math.Min(longitud, texto.Length - inicio)`.

**4. Llamar métodos sobre `null`.**
Qué pasa: `NullReferenceException`.
Por qué: un string `null` no tiene métodos.
Arreglo: `string.IsNullOrWhiteSpace(texto)` antes, o `texto?.Trim()`.

**5. Escribir `Length()` con paréntesis.**
Qué pasa: `error CS1955: Non-invocable member 'string.Length' cannot be used like a method`.
Por qué: `Length` es una propiedad, no un método.
Arreglo: `texto.Length`.

**6. Concatenar en un bucle grande con `+=`.**
Qué pasa: el programa se vuelve lento y consume mucha memoria con miles de iteraciones.
Por qué: cada `+=` copia todo el texto acumulado.
Arreglo: `StringBuilder`, o `string.Join` si ya tienes los elementos en una colección.

-----

## Según la versión de C#

* **C# 8:** índices y rangos: `texto[^1]`, `texto[2..5]`, `texto[..3]`.
* **.NET Core 2.0+:** sobrecargas de `Contains`, `Replace`, `Split` y `StartsWith` que reciben `char` o `StringComparison`.
* **.NET 5:** `StringSplitOptions.TrimEntries`.
* **C# 10 / .NET 6:** `StringBuilder.Append($"...")` usa un manejador de interpolación que escribe directo en el búfer, sin crear el string intermedio.

-----

## Cuándo sí y cuándo no

**Usa los métodos de `string` cuando:**

* Haces pocas operaciones sobre textos cortos. Es lo más legible.

**Usa `StringBuilder` cuando:**

* Construyes texto dentro de un bucle o en muchos pasos (reportes, HTML, archivos).
* Como referencia: a partir de unas pocas decenas de concatenaciones en un bucle ya se nota.

**No uses `StringBuilder` cuando:**

* Unes 2 o 3 piezas: `$"{a} {b}"` es más claro y no es más lento.
* Ya tienes una colección: `string.Join(", ", lista)` lo resuelve en una línea.

-----

## Resumen en 5 líneas

1. `Length`, `IndexOf` (-1 si no existe), `Contains`, `StartsWith` y `EndsWith` consultan.
2. `Substring(inicio, longitud)` y los rangos `[inicio..fin]` extraen partes.
3. `Trim`, `ToUpper`, `Replace`, `PadLeft`, `Split` y `string.Join` transforman y devuelven un string nuevo.
4. Para comparar sin distinguir mayúsculas, usa `StringComparison.OrdinalIgnoreCase`, no `ToLower()`.
5. `StringBuilder` construye texto en muchos pasos sin crear un string por cada paso.

-----

## Para profundizar

<details>
<summary>IndexOf(string) depende de la cultura</summary>

`IndexOf(char)` compara de forma ordinal, pero `IndexOf(string)` sin más argumentos usa la **cultura actual**. Desde .NET 5 (que usa la librería ICU), eso puede dar resultados sorprendentes con caracteres invisibles o combinados, por ejemplo con `"\r\n"`. En código que procesa datos, indica la comparación explícitamente:

```csharp
int pos = texto.IndexOf("\n", StringComparison.Ordinal);
```

Los analizadores de código de .NET (regla CA1310) te avisan cuando falta.

</details>

<details>
<summary>Span y rendimiento</summary>

`Substring` y `Split` crean strings nuevos en el heap. En código de alto rendimiento (parsers, servidores) se usa `ReadOnlySpan<char>`, una "ventana" sobre el texto original que no copia nada:

```csharp
ReadOnlySpan<char> dominio = correo.AsSpan(arroba + 1);
```

No lo necesitas para empezar, pero lo verás en código de librerías.

</details>

<details>
<summary>Expresiones regulares (Regex)</summary>

Cuando los métodos de string no alcanzan (validar formatos, extraer patrones), se usan expresiones regulares con `System.Text.RegularExpressions.Regex`:

```csharp
using System.Text.RegularExpressions;

bool esCodigo = Regex.IsMatch("ABC-1234", @"^[A-Z]{3}-\d{4}$");             // True
string soloDigitos = Regex.Replace("Tel: (01) 555-1234", @"\D", "");       // "015551234"
foreach (Match m in Regex.Matches("a1 b22 c333", @"\d+"))
    Console.Write($"{m.Value} ");                                            // 1 22 333
```

Son potentes pero difíciles de leer: úsalas cuando de verdad simplifican, y deja un comentario con lo que valida el patrón. Documentación: [Regex](https://learn.microsoft.com/dotnet/api/system.text.regularexpressions.regex).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los strings tienen métodos como `IndexOf`, `Substring`, `Replace`, `Trim`, `Split` o `ToUpper`, y todos devuelven un string nuevo porque los strings son inmutables. Para construir texto en un bucle se usa `StringBuilder`, que evita crear un string en cada concatenación.

### Respuesta ampliada (semi-senior)

Concatenar con `+=` en un bucle es O(n²) en copias, porque cada iteración copia el texto acumulado; `StringBuilder` mantiene un búfer de bloques que crece y solo materializa el string con `ToString()`. Para pocos fragmentos, la interpolación es igual de eficiente y más legible, y `string.Join`/`string.Concat` cubren las colecciones. En las comparaciones conviene ser explícito con `StringComparison`: `Ordinal` u `OrdinalIgnoreCase` para datos y claves, y cultura solo para texto visible. En código de alto rendimiento se trabaja con `ReadOnlySpan<char>` para no crear subcadenas.

### Preguntas frecuentes de seguimiento

**1. ¿Cuándo usar `StringBuilder` en lugar de `+`?**
Cuando concatenas muchas veces, sobre todo dentro de un bucle. Para 2 o 3 piezas no hace falta.

**2. ¿Qué devuelve `IndexOf` si no encuentra el texto?**
`-1`.

**3. ¿Por qué no comparar con `ToLower()`?**
Porque crea strings nuevos innecesarios y, según la cultura, puede dar resultados incorrectos (el caso clásico es la "i" turca). `StringComparison.OrdinalIgnoreCase` es correcto y más eficiente.

-----

## Práctica

**Ejercicio 1.** Dado `string nombreCompleto = "  torres, ana  ";`, obtén `"Ana Torres"` usando métodos de string.

<details>
<summary>Solución</summary>

```csharp
string nombreCompleto = "  torres, ana  ";

string[] partes = nombreCompleto.Split(',', StringSplitOptions.TrimEntries);
string apellido = partes[0];
string nombre = partes[1];

string Capitalizar(string s) => char.ToUpper(s[0]) + s[1..];

Console.WriteLine($"{Capitalizar(nombre)} {Capitalizar(apellido)}");   // Ana Torres
```

`TrimEntries` quita los espacios de cada parte. `Capitalizar` es una función local; las funciones se ven en el módulo de [Métodos](../03-metodos/README.md).

</details>

**Ejercicio 2.** Usa `StringBuilder` para generar la tabla de multiplicar del 7 (del 1 al 10), con el formato `7 x 3 = 21`, y muéstrala de una sola vez.

<details>
<summary>Solución</summary>

```csharp
using System.Text;

var sb = new StringBuilder();
for (int i = 1; i <= 10; i++)
{
    sb.AppendLine($"7 x {i,2} = {7 * i,2}");
}
Console.Write(sb);   // Write llama a ToString() automáticamente
```

</details>

-----

## Siguiente lección

Terminaste el módulo de tipos y variables. Continúa con [Control de flujo](../02-control-de-flujo/README.md).
